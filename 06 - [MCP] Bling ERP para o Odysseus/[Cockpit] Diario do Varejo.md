---
name: cockpit-diario-varejo
description: "Cockpit Diário do Varejista via MCP Bling — briefing operacional consolidado em uma única execução, com vendas de ontem/hoje por canal, pedidos travados na expedição, alertas de estoque crítico e compromissos financeiros do dia. Use sempre que o usuário pedir \"bom dia\", briefing, resumo do dia, panorama, cockpit, \"como estamos hoje\", visão geral da operação, ou um status rápido da loja — mesmo sem citar a skill."
---

# Cockpit Diário do Varejista

Um único comando, quatro painéis. Pensado para abrir o dia em menos de 2 minutos de leitura.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listAccountPayables listAccountReceivables listProducts getStockBalance listNfes listStatusModules"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder` (amostral), `listAccountPayables`, `listAccountReceivables`, `listProducts`, `getStockBalance`, `listNfes`, `listStatusModules`.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:cockpit-diario-varejo] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Os 4 painéis (executar nesta ordem)

### Painel 1 — Vendas (ontem + hoje)
`listSalesOrders(dataInicial: ontem, dataFinal: hoje, fields: ["id","numero","data","total","loja","situacao"])`
→ Faturamento, nº de pedidos, ticket médio, quebra por canal (`loja.id`), comparação com a média diária dos últimos 7 dias (segunda chamada). Destacar o maior pedido do dia.

### Painel 2 — Expedição
Dos pedidos não cancelados das últimas 48h, verificar via `getSalesOrder` (**teto de 40 chamadas**, priorizando os mais antigos) os que têm `notaFiscal.id = 0` (não faturados) ou volume sem `codigoRastreamento` (não postados). Se a fila passar de 40 pedidos, os totais do painel são da parcela verificada: dizer "pelo menos N pedidos aguardando NF" em vez de apresentar como contagem fechada.
→ "X pedidos aguardando NF, Y aguardando postagem, valor represado R$ Z". Para análise profunda, indicar a skill **sentinela-expedicao**.

### Painel 3 — Estoque crítico
`listProducts(filtroSaldoEstoque: "2", fields: ["id","nome","codigo","estoque"])` para saldos negativos (erro de lançamento) e `getStockBalance` dos top vendidos da semana para cobertura.
→ Lista curta: até 5 SKUs em risco. Para o estudo completo, indicar a skill **radar-ruptura-estoque**.

### Painel 4 — Financeiro do dia
`listAccountPayables(dataVencimentoInicial: hoje, dataVencimentoFinal: amanhã)` e `listAccountReceivables(tipoFiltroData: "V", dataInicial: hoje, dataFinal: amanhã, situacoes: [1, 3])`.

Os dois endpoints filtram data de formas diferentes: contas a pagar tem `dataVencimentoInicial`/`dataVencimentoFinal` próprios; contas a receber usa `dataInicial`/`dataFinal` com `tipoFiltroData: "V"`, porque sem isso o padrão é filtrar por **emissão** e o painel mostraria títulos emitidos hoje em vez de vencendo hoje.
→ "Hoje vencem R$ X a pagar e R$ Y a receber; saldo líquido do dia: ±R$ Z". Destacar título vencido em aberto, se houver.

## Formato de saída

```
☀️ COCKPIT — <data>

💰 VENDAS      ontem R$ … (N pedidos, ticket R$ …) | Δ vs média 7d: ±%
               por canal: Meli R$… · Shopee R$… · Site R$…
📦 EXPEDIÇÃO   X sem NF · Y sem postagem · R$ Z represados
📉 ESTOQUE     N SKUs críticos: <lista curta>
🏦 FINANCEIRO  pagar hoje R$ … · receber hoje R$ … · líquido ±R$ …

⚡ 3 PRIORIDADES DO DIA
1. …
2. …
3. …
```

Manter o briefing em uma tela; profundidade fica nas skills especializadas. Se algum painel falhar (tool indisponível), reportar o painel como "sem dados" e seguir com os demais — nunca abortar o briefing inteiro.