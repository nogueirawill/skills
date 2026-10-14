---
name: auditor-canais-marketplace
description: "Auditoria de Performance por Canal de Venda (marketplaces e loja virtual) via MCP Bling. Agrupa pedidos por loja/canal (Mercado Livre, Shopee, Amazon, e-commerce próprio), calcula faturamento, ticket médio, share de vendas e detecta quedas bruscas por canal. Use sempre que o usuário mencionar marketplaces, canais de venda, comparação entre lojas, ticket médio, share de faturamento, \"qual canal vende mais\", performance da Shopee/Meli/Amazon, ou pedir \"Auditoria de Canais\"."
---

# Auditor de Performance por Canal (Marketplaces)

Compara rentabilidade e volume entre canais de venda integrados ao Bling.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listStatusModules"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listStatusModules` (mapear situações).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:auditor-canais-marketplace] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Conceito-chave: identificação do canal

No Bling, o canal aparece no pedido pelos campos:
- `loja.id` — ID da integração/loja (cada marketplace conectado tem um ID)
- `numeroLoja` — número do pedido na plataforma de origem (vazio = venda direta/manual)
- `intermediador.cnpj` — CNPJ do marketplace (ex.: Mercado Livre = 03.007.331/0001-41) — use como confirmação do canal quando o nome da loja não for óbvio
- `taxas.taxaComissao` e `transporte.frete` — custos por canal

## Fluxo de execução

### 1. Coletar pedidos do período (padrão: últimos 7 dias; aceitar período do usuário)
```
listSalesOrders(dataInicial, dataFinal,
                fields: ["id","numero","data","total","totalProdutos","loja","situacao"])
```
Para subconjuntos por canal conhecido, usar `idLoja`.

**Comportamento padrão**: em prompts mínimos ("Análise de canais", "canais"), tratar sempre como análise de performance por canal de marketplace — usar `listSalesOrders` com agrupamento por `intermediador.cnpj`/`loja.id`, não `listProducts`. Incluir comparação com o período anterior equivalente por padrão.

### 2. Detalhar custos por canal (amostra)
Para os **3 maiores pedidos de cada `loja.id`** (máx. 15 chamadas totais), chamar `getSalesOrder` e extrair `taxas.taxaComissao`, `frete` e `intermediador` para nomear o canal e estimar margem após comissão.

### 3. Calcular por canal
- Faturamento total e nº de pedidos
- **Ticket médio** = faturamento ÷ pedidos
- **Share** = faturamento do canal ÷ faturamento total
- **Comissão média %** (amostra) e faturamento líquido estimado
- **Variação vs. período anterior equivalente** (rodar segunda chamada de `listSalesOrders` para o período anterior): queda > 30% no canal = ALERTA

### 4. Relatório (formato de saída)
1. Tabela-ranking: Canal | Pedidos | Faturamento | Ticket médio | Share % | Δ vs. período anterior
2. Gráfico de pizza do share de vendas (widget/artifact)
3. Alertas: canais em queda brusca, canais com comissão acima da média, pedidos sem canal identificado (venda manual)
4. 3 recomendações acionáveis (ex.: reforçar anúncios no canal X, revisar precificação no canal Y dada a comissão)

**Nota sobre margens**: valores de margem são estimativas baseadas nos dados de custo disponíveis no Bling e podem não refletir taxas reais de marketplace ou comissões de plataforma.

## Cuidados
- Excluir pedidos cancelados do faturamento. Pedido de venda usa IDs dinâmicos por conta: seguir o procedimento de discovery do `bling://guia/situacoes` e reutilizar o mapa pelo resto da sessão. Como esta é uma análise de faturamento (consulta de histórico/ranking), valem as regras de apresentação do guia para esse tipo — considerar apenas situações concluídas.
- Se `loja.id` = 0 ou ausente, classificar como "Venda direta / manual".
- Períodos > 365 dias exigem confirmação da API — preferir janelas curtas e comparativas.