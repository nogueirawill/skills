---
name: vigia-fiscal-nfe
description: "Vigia Fiscal de NF-e/NFC-e via MCP Bling. Audita pedidos faturáveis sem nota emitida, notas pendentes/rejeitadas/denegadas na Sefaz e mede aderência fiscal por canal. Use sempre que o usuário mencionar nota fiscal, NF-e, NFC-e, Sefaz, notas rejeitadas, pedidos sem nota, emissão de nota, compliance fiscal, ou perguntar \"quais pedidos têm nota fiscal emitida\"."
---

# Vigia Fiscal (NF-e / NFC-e)

Auditoria do faturamento fiscal: o que deveria ter nota e não tem, e o que travou na Sefaz.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listNfes listNfces getNfe listSalesOrders getSalesOrder listPurchaseOrders getPurchaseOrder generateSalesOrderNfe authorizeNfe"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listNfes`, `listNfces`, `getNfe`, `listSalesOrders`, `getSalesOrder`, `listPurchaseOrders`, `getPurchaseOrder`, `generateSalesOrderNfe` e `authorizeNfe` (escrita — só com confirmação explícita).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:vigia-fiscal-nfe] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Situações de NF-e (parâmetro `situacao` de listNfes)
`1` Pendente · `2` Cancelada · `3` Aguardando recibo · `4` Rejeitada · `5` Autorizada · `6` Emitida DANFE · `7` Registrada · `8` Aguardando protocolo · `9` Denegada · `11` Bloqueada.
Sem o parâmetro, canceladas não entram na consulta. `tipo`: `0` entrada, `1` saída.

## Sinal-chave nos pedidos
Em **pedido de venda** (`getSalesOrder`), `notaFiscal.id = 0` significa **sem NF vinculada** — é o teste definitivo de pedido não faturado.

Em **pedido de compra** o campo não existe na raiz: a nota aparece por item, em `itens[].notaFiscal.id`. Entrada sem nota é o pedido em que nenhum item tem `notaFiscal.id` preenchido. Aplicar o teste da raiz em compras retorna vazio para todo pedido e classificaria 100% das compras como sem lastro fiscal.

## Fluxo de execução

### 1. Levantar notas do período (janela padrão: últimos 30 dias)
`listNfes(tipo: "1", dataEmissaoInicial, dataEmissaoFinal, buscar_tudo: true, fields: ["id","numero","situacao","dataEmissao","contato"])`.

O parâmetro `situacao` de `listNfes` aceita **um valor por chamada** (é inteiro, não array), diferente de `situacoes[]` de contas a receber ou `idsSituacoes[]` de pedidos de venda. Então é uma chamada por situação problemática: `4` (Rejeitada), `9` (Denegada), `11` (Bloqueada), `1` (Pendente). Se a operação usa PDV, repetir com `listNfces`.

### 2. Cruzar com pedidos
`listSalesOrders(buscar_tudo: true)` do mesmo período para saber o universo, e então `getSalesOrder` nos não cancelados para verificar `notaFiscal.id = 0`.

**Teto de 150 `getSalesOrder` por execução** (3 requisições por segundo). Se o período tiver mais pedidos que isso, priorizar os mais antigos e de maior valor — são os de maior risco fiscal — e declarar no relatório quantos pedidos foram verificados de quantos existem. O funil fiscal e o "valor total em risco" saem sobre os pedidos verificados, não sobre o período todo: apresentar como tal, ou reduzir a janela para obter cobertura total. Isso importa porque esse número é a base da decisão de emitir nota no passo 5.

Em compras, mesmo cruzamento com `listPurchaseOrders`/`getPurchaseOrder`, com o mesmo teto e usando `itens[].notaFiscal.id` conforme descrito acima.

### 3. Diagnóstico de travas
Para notas rejeitadas/denegadas, `getNfe(idNotaFiscal)` e resumir o motivo em linguagem simples (erro de cadastro, NCM, destinatário etc.).

### 4. Relatório (formato de saída)
1. **Funil fiscal**: pedidos do período → com NF → autorizadas (% em cada etapa)
2. Lista de pedidos faturáveis sem NF (pedido, canal, valor, idade)
3. Lista de NFs travadas (número, situação, motivo resumido, ação sugerida)
4. Aderência por canal (marketplace X com N% de pedidos sem nota)
5. Valor total em risco fiscal

### 5. Ações opcionais (somente com confirmação explícita, uma a uma)
- `generateSalesOrderNfe(idPedidoVenda)` para gerar a NF do pedido
- `authorizeNfe(idNotaFiscal)` para enviar à Sefaz
Nunca executar em lote sem listar antes exatamente o que será emitido. Questões tributárias específicas (CFOP, ST, regime) devem ser confirmadas com o contador do usuário.
