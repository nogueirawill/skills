---
name: esteira-faturamento-guiado
description: "Esteira de Faturamento Guiado via MCP Bling. Processa em lote os pedidos de venda aprovados: pré-valida estoque e cadastro fiscal (NCM/GTIN/origem), triagem de aptos vs. bloqueados, e — com confirmação explícita — gera e autoriza as NF-e, baixa estoque e atualiza status. Use sempre que o usuário pedir para faturar pedidos, emitir notas em lote, \"rodar a esteira\", processar pedidos em aberto, faturamento do dia, ou quiser emitir NF-e de vários pedidos de uma vez."
---

# Esteira de Faturamento Guiado

Recebe a fila corrigida do preflight-fiscal-natureza/curador-catalogo e a converte em NF-e: faturamento em lote com pré-validação, primeiro triagem, depois emissão — nunca o contrário. O objetivo é eliminar o ciclo "emite → Sefaz rejeita → corrige → reemite".

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder getStockBalance getProduct generateSalesOrderNfe authorizeNfe manageSalesOrderStock updateSalesOrderStatus listStatusModules listStatusModuleTransitions"` (limit: 25). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `getStockBalance`, `getProduct`, `generateSalesOrderNfe`, `authorizeNfe`, `manageSalesOrderStock`, `updateSalesOrderStatus`, `listStatusModules`, `listStatusModuleTransitions`.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:esteira-faturamento-guiado] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Levantar a fila
`listSalesOrders` do período (**padrão: últimos 7 dias**; aceitar período do usuário se especificado) filtrando pedidos não cancelados. Para cada candidato, `getSalesOrder` e separar os que têm `notaFiscal.id = 0` (sem NF vinculada) — esses formam a fila de faturamento.

**Teto de 150 `getSalesOrder` por execução.** Se a fila do período for maior, processar os mais antigos primeiro (maior risco de SLA) e informar quantos pedidos ficaram fora desta rodada, oferecendo rodar a esteira novamente para o restante. Nunca dar a fila como completa quando o teto foi atingido: o usuário precisa saber que sobrou pedido a faturar.

### 2. Pré-validação de estoque
`getStockBalance` dos SKUs da fila. Bloquear pedidos cujos itens deixariam o saldo negativo. Pedido bloqueado por estoque nunca avança para emissão — faturar sobre saldo inexistente propaga erro contábil e fiscal.

### 3. Pré-validação de cadastro fiscal
`getProduct` de cada item da fila. Verificar campos que causam rejeição na Sefaz: NCM ausente/inválido, GTIN/EAN vazio quando exigido, origem da mercadoria, unidade. Pedido com item incompleto vai para a fila de bloqueados com o campo faltante nomeado.

### 4. Quadro de triagem (sempre antes de emitir)
Apresentar três listas: **aptos** (pedido, canal, cliente, UF, valor), **bloqueados por estoque** (item e saldo), **bloqueados por cadastro** (item e campo faltante). Informar qual Natureza de Operação está associada a cada pedido apto e destacar pedidos interestaduais.

### 5. Emissão em lote (somente após confirmação explícita do usuário)
A confirmação deve ser uma pergunta direta nomeando os pedidos específicos: ex. "Posso emitir NF-e para os pedidos #123, #124 e #125?" — aguardar sim/não antes de chamar `generateSalesOrderNfe`.

Para cada pedido confirmado, nesta ordem:

1. `generateSalesOrderNfe(idPedidoVenda)` — gera a NF a partir do pedido
2. `authorizeNfe(idNotaFiscal)` — envia à Sefaz
3. `manageSalesOrderStock` — baixa de estoque, se ainda não vinculada ao fluxo
4. `updateSalesOrderStatus` — mover para o status pós-faturamento (usar `listStatusModules`/`listStatusModuleTransitions` para descobrir os IDs válidos de situação da conta, nunca chutar)

Nunca autorizar NF sem listar antes exatamente o que será emitido. Se qualquer autorização retornar rejeição, interromper o lote, reportar o motivo em linguagem simples e não tentar reemissão automática. Em caso de rejeição, indicar o campo problemático e sugerir correção via preflight-fiscal-natureza antes de nova tentativa.

### 6. Relatório final
Funil do lote (fila → aptos → emitidas → autorizadas), pedidos que falharam com motivo, e pendências de cadastro para corrigir antes da próxima rodada.

## Cuidados
A tributação vem do cálculo automático da Natureza de Operação configurada no Bling — a skill não define alíquota, CFOP ou CST. Se um pedido interestadual ou de devolução parecer usar natureza de venda interna, perguntar ao usuário antes de emitir. Dúvidas de regime tributário (Simples vs. Normal, ST) devem ser confirmadas com o contador.
