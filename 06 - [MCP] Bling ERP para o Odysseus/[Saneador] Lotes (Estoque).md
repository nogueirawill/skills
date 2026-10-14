---
name: saneador-estoque-lotes
description: "Saneador de Estoque, Lotes e Depósitos via MCP Bling. Cuida da integridade do estoque: detecta saldos negativos com hipótese de causa, desequilíbrio entre depósitos (sobra em um, ruptura em outro), lotes próximos do vencimento (30/60/90 dias) e propõe lançamentos de ajuste com aprovação item a item. Use sempre que o usuário mencionar saldo negativo, estoque errado, divergência de estoque, ajuste de inventário, lotes, validade, vencimento de produto, transferência entre depósitos, ou pedir para \"sanear o estoque\"."
---

# Saneador de Estoque, Lotes e Depósitos

Integridade, não reposição: enquanto o radar-ruptura-estoque calcula cobertura e sugestão de compra, esta skill encontra e corrige o estoque **errado** — negativos, desequilíbrios entre depósitos e lotes vencendo.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listWarehouses getStockBalance listProductBatches listSalesOrders listPurchaseOrders getPurchaseOrder createStock"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listWarehouses`, `getStockBalance`, `listProductBatches`, `listSalesOrders`, `listPurchaseOrders`, `getPurchaseOrder`, `createStock` (escrita — só com aprovação item a item).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:saneador-estoque-lotes] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Mapa de depósitos
`listWarehouses` → relação de depósitos ativos e qual é o padrão.

### 2. Varredura de saldos
`getStockBalance` dos produtos (por depósito quando disponível). Classificar:
- **Negativos**: saldo < 0 em qualquer depósito
- **Desequilibrados**: SKU com sobra num depósito e saldo zero/negativo em outro
- **Fantasmas**: saldo total positivo mas indisponível para venda no depósito que atende os canais

### 3. Hipótese de causa dos negativos
Cruzar cada SKU negativo com `listSalesOrders` recentes (venda sem baixa correspondente?) e `listPurchaseOrders`/`getPurchaseOrder` (entrada recebida mas não lançada?). Reportar a hipótese mais provável por SKU — nunca ajustar sem explicar a causa provável.

**Teto de 100 `getPurchaseOrder` por execução**, priorizando os SKUs com saldo mais negativo (maior impacto). SKU não investigado entra no relatório como "causa não apurada", nunca com hipótese inventada.

### 4. Lotes e validade (obrigatório)
`listProductBatches` — chamada **obrigatória em toda execução**. Se retornar HTTP 403, registrar "acesso negado ao módulo de lotes — verificar escopo OAuth" e continuar com as demais seções. Estruturar o relatório em três buckets explícitos:
- **Vencendo em 30 dias**: produto, lote, quantidade, data de vencimento
- **Vencendo em 31–60 dias**: idem
- **Vencendo em 61–90 dias**: idem
Se nenhum lote vencer em um bucket, indicar "sem lotes vencendo neste período". Sinalizar lote vencido ainda com saldo (risco sanitário e fiscal) e sugerir ação comercial (queima, kit) para janela de 60–90 dias.

### 5. Proposta de ajustes (escrita somente com aprovação item a item)
Para cada correção aprovada pelo usuário individualmente, `createStock` com a operação adequada (entrada de acerto, saída de acerto) e observação registrando o motivo. Nunca executar ajustes em lote sem listar antes cada lançamento com produto, depósito, quantidade e justificativa.

### 6. Relatório
1. Saldos negativos: SKU, depósito, quantidade, hipótese de causa, ajuste proposto
2. Desequilíbrios entre depósitos com sugestão de transferência
3. Calendário de vencimento de lotes com valor em risco (Se `listProductBatches` retornou 403, esta seção exibe: [acesso negado ao módulo de lotes — verificar escopo OAuth])
4. Ajustes executados (se houver) com trilha do que foi lançado

## Cuidados
Ajuste de estoque pode ter reflexo fiscal: transferência entre filiais e baixa por perda/perecimento usam CFOPs e Naturezas de Operação próprios. Quando a movimentação exigir documento fiscal, a skill não lança o acerto "seco" — sinaliza que a operação deve ser acobertada por nota com a natureza correta e pergunta ao usuário como proceder. Saldo negativo é bloqueador de faturamento (integração com a esteira-faturamento-guiado).
