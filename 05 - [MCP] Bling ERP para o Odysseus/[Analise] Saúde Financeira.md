---
name: saude-financeira-bling
description: "Analista de Saúde Financeira (DRE operacional) via MCP Bling. Consolida contas a pagar e a receber, projeta fluxo de caixa dos próximos 15 dias, calcula inadimplência e destaca as maiores despesas e recebimentos previstos. Use sempre que o usuário mencionar fluxo de caixa, contas a pagar, contas a receber, inadimplência, saldo projetado, boletos, DRE, resumo financeiro, \"quanto entra/sai\", ou pedir o \"Resumo Financeiro\"."
---

# Analista de Saúde Financeira (DRE Operacional)

Visão rápida de caixa: o que entra, o que sai e onde está o risco.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listAccountPayables listAccountReceivables getAccountReceivableBoletos listSalesOrders getContact"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listAccountPayables`, `listAccountReceivables`, `getAccountReceivableBoletos`, `listSalesOrders` (faturamento do período), `getContact` (identificar credores/devedores).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:saude-financeira-bling] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

Não repetir a mesma consulta com filtros idênticos — reutilizar o resultado já obtido na sessão em vez de chamar `listAccountReceivables` ou `listAccountPayables` de novo com os mesmos parâmetros. Duas exceções legítimas: **paginação** (avançar `pagina` para completar o conjunto de dados) e **mudança de janela pedida pelo usuário** (outro horizonte, outro intervalo de vencimento).

## Fluxo de execução

### 1. Contas a receber
```
listAccountReceivables(tipoFiltroData: "V", dataInicial: hoje, dataFinal: hoje+15d,
                       buscar_tudo: true)
```

**`tipoFiltroData: "V"` é obrigatório**: o padrão da API é `E` (data de emissão), e uma projeção de caixa feita sobre data de emissão não projeta nada — o que importa é quando o dinheiro entra, ou seja o vencimento. Rodar uma segunda consulta para o passado recente (vencidas e não recebidas) para medir inadimplência.

Separar por situação usando o enum de contas a receber (`1` Em aberto, `2` Recebido, `3` Parcialmente recebido, `4` Devolvido, `5` Cancelado): só `1` e `3` entram como entrada prevista; `2` já entrou; `4` e `5` não entram nunca. Para títulos relevantes em aberto, `getAccountReceivableBoletos(idOrigem)` confirma boletos vinculados e status.

### 2. Contas a pagar
`listAccountPayables(dataVencimentoInicial: ..., dataVencimentoFinal: ..., buscar_tudo: true)` no mesmo horizonte. Aqui não existe `tipoFiltroData`: este endpoint tem parâmetros de vencimento próprios, então usar `dataVencimentoInicial`/`dataVencimentoFinal` em vez de `dataInicial`/`dataFinal`, que filtram por emissão. Identificar as 5 maiores despesas previstas e despesas recorrentes. Resolver nomes de fornecedores via `getContact` quando vier só o ID.

### 3. Projeção de caixa (dia a dia)
Para cada dia D do horizonte: `saldo(D) = saldo(D-1) + recebimentos previstos(D) − pagamentos previstos(D)`.
- Se o usuário não informar o saldo inicial, perguntar UMA vez; se não souber, projetar a partir de zero (saldo inicial = 0 para o primeiro dia do horizonte) e deixar claro que é fluxo líquido, não saldo bancário.
- Marcar o **primeiro dia de caixa negativo** (ponto de atenção).

### 4. Indicadores
- **Inadimplência %** = vencidas em aberto ÷ total a receber do período
- **Cobertura** = total a receber ÷ total a pagar no horizonte (≥ 1,2 saudável; < 1 alerta)
- **Concentração**: % do recebível concentrado nos 3 maiores clientes/canais
- **Faturamento vs. recebimento**: comparar `listSalesOrders` do período com entradas previstas (vendas de marketplace têm repasse defasado)

### 5. Relatório (formato de saída)
1. Tabela diária: Data | Entradas | Saídas | Líquido do dia | Acumulado
2. Top 5 despesas e Top 5 recebimentos do período
3. Inadimplência: lista de títulos vencidos com cliente, valor e dias de atraso
4. Semáforo final (🟢/🟡/🔴) com justificativa e **3 ações concretas** — cada ação deve nomear valor específico e data de vencimento (ex.: "antecipar cobrança de R$ 1.200 do cliente X com vencimento em DD/MM", "renegociar despesa de R$ 800 do fornecedor Y prevista para DD/MM")
5. Se solicitado, gráfico de linha do acumulado via widget

## Cuidados
- Esta análise é informativa e operacional; não é aconselhamento financeiro formal — decisões de crédito/investimento exigem o contexto completo do usuário.
- Nunca criar/alterar lançamentos (`createAccountPayable`/`createAccountReceivable`) sem instrução explícita.
- Datas no formato AAAA-MM-DD; janelas curtas para evitar paginação excessiva.