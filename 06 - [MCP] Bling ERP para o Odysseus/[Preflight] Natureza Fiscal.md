---
name: preflight-fiscal-natureza
description: "Pré-Voo Fiscal via MCP Bling. Validação preventiva antes da emissão: cruza cadastro de produtos (NCM, CEST, GTIN, origem) com o histórico de rejeições da Sefaz e com as UFs de destino reais dos pedidos, gerando fila priorizada de correção de cadastro e mapa de risco por Natureza de Operação. Use sempre que o usuário mencionar pré-voo fiscal, validar cadastro fiscal, NCM, CEST, produtos prontos para emitir, prevenção de rejeição Sefaz, natureza de operação, CFOP, ou perguntar \"por que minhas notas são rejeitadas\"."
---

# Pré-Voo Fiscal (Natureza de Operação)

Prevenção, não auditoria: identifica o que **vai** falhar na Sefaz antes de qualquer emissão. Complementa a vigia-fiscal-nfe (que olha o que já falhou).

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listProducts getProduct listProductVariations listNfes getNfe listSalesOrders getContact"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listProducts`, `getProduct`, `listProductVariations`, `listNfes`, `getNfe`, `listSalesOrders`, `getContact`. Skill 100% de leitura — nenhuma escrita.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:preflight-fiscal-natureza] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Matriz de completude fiscal do catálogo
`listProducts` com `tipo=P` (produtos físicos) e `criterio=2` (Ativos) — não buscar catálogo inteiro sem filtro, e não confiar no padrão de `criterio`, que é `1` (Últimos incluídos) e deixaria produtos antigos fora da validação fiscal. **Teto de paginação: no máximo 5 páginas.** Como a listagem retorna uma página por chamada, isso significa chamar `listProducts` incrementando `pagina` enquanto `has_more` for `true`, parando em `pagina=5` (até 500 produtos). Não usar `buscar_tudo` aqui: a validação fiscal é feita por amostra priorizada, não exige o catálogo inteiro. Ao atingir o teto, proceder com o que foi obtido e indicar "análise parcial: N produtos analisados". Para os mais vendidos ou para amostra definida pelo usuário, `getProduct` (**teto de 100 chamadas por execução**, priorizando os SKUs com maior volume de venda no período) e avaliar: NCM (presente e com 8 dígitos), CEST (quando o NCM indicar ST), GTIN/EAN, origem da mercadoria, unidade, descrição. Para produtos com variações, `listProductVariations` — variação herda ou quebra o cadastro do pai.

### 2. Destinos reais da operação
`listSalesOrders` dos últimos 30–60 dias → distribuição de UFs de destino por canal. Para validação CFOP interestadual, usar `getContact` com o `contato.id` do pedido para resolver a UF real do comprador — não assumir UF apenas pelos dados do pedido. Isso define quais operações interestaduais a Natureza de Operação precisa cobrir de verdade (não adianta natureza perfeita para SP se 40% das vendas vão para o Nordeste).

### 3. Histórico de rejeição como evidência
`listNfes` com situações 4 (Rejeitada) e 9 (Denegada) do período. Para cada uma, `getNfe` e correlacionar o código/motivo de rejeição com o campo de cadastro correspondente (ex.: rejeição por NCM inválido → produto X, campo NCM).

### 4. Mapa de risco e fila de correção
Saída em duas partes:
1. **Mapa SKU × UF × risco**: quais produtos, vendidos para quais estados, têm maior probabilidade de rejeição
2. **Fila priorizada de correção**: ordenada por volume de vendas × gravidade do campo faltante — o SKU campeão de vendas com NCM vazio vai para o topo

### 5. Checklist de Natureza de Operação
Para cada tipo de operação detectada nos pedidos (venda interna, interestadual, consumidor final, devolução), listar o que a natureza correspondente precisa cobrir e sinalizar lacunas prováveis. A skill **não** cria nem edita naturezas (isso é feito no painel do Bling) — ela entrega o diagnóstico pronto para o usuário ou o contador configurar.

## Cuidados
O cálculo automático do Bling depende de a Natureza de Operação estar configurada e de o cadastro do produto estar completo — regra específica por estado/produto falha silenciosamente se o produto não tem os campos preenchidos. Em Regime Normal, alíquotas e CST são definição do contador; a skill aponta o campo ausente, nunca sugere valor de tributo.
