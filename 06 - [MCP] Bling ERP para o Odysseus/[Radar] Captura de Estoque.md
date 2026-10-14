---
name: radar-ruptura-estoque
description: "Radar de Ruptura e Giro de Estoque via MCP Bling. Cruza vendas recentes com saldo de estoque para calcular cobertura (runway), gerar lista de reposição prioritária e identificar capital parado (produtos mortos). Use sempre que o usuário mencionar ruptura, reposição, giro de estoque, cobertura de estoque, \"vai acabar\", produtos parados, estoque negativo, sugestão de compra, ou pedir o \"Radar de Ruptura\" — mesmo que não cite a skill pelo nome."
---

# Radar de Ruptura e Giro de Estoque

Skill para prever ruptura de estoque e identificar capital parado na conta Bling do usuário.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listProducts getStockBalance listProductSuppliers createPurchaseOrder"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listProducts`, `getStockBalance`, `listProductSuppliers`, `createPurchaseOrder` (opcional, só com confirmação explícita).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:radar-ruptura-estoque] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Coletar vendas do período (padrão: últimos 30 dias)
```
listSalesOrders(dataInicial: "AAAA-MM-DD", dataFinal: "AAAA-MM-DD",
                fields: ["id","numero","data","total"])
```
- Sempre informe `dataInicial`/`dataFinal`. Períodos > 365 dias exigem confirmação extra da API — evite.
- Use `buscar_tudo: true` nesta listagem: é preciso saber **quantos pedidos existem** no período para julgar se a análise é confiável. Isso é barato (só a listagem, sem itens).
- A listagem NÃO traz itens. Para obter os SKUs vendidos, é necessário `getSalesOrder(idPedidoVenda)` por pedido. A 3 requisições por segundo, **teto de 150 `getSalesOrder` por execução** (~50 s).

**Decidir o modo de análise antes de seguir**, conforme o total de pedidos do período:

- **Pedidos ≤ 150** → cobertura total. Todos os indicadores são confiáveis, inclusive "produto morto".
- **Pedidos > 150** → cobertura parcial. Detalhar os 150 mais recentes e **avisar o usuário antes de apresentar números**: a demanda medida é de uma parte do período. Nesse modo, a recomendação é reduzir a janela (ex.: 15 dias em vez de 30) ou filtrar por categoria/canal, o que devolve cobertura total sobre um recorte menor e é bem mais útil que uma média sobre pedaço arbitrário.

Registrar sempre no relatório: pedidos no período, pedidos detalhados e qual modo foi usado.

### 2. Agregar demanda por produto
Para cada item dos pedidos: somar `quantidade` por `produto.id` / `codigo`. Calcular **venda média diária** = total vendido ÷ dias do período.

### 3. Consultar estoque atual
```
getStockBalance(idsProdutos: [ids dos produtos vendidos])
```
Complementar com `listProducts(fields: ["id","nome","codigo","preco","estoque"], criterio: "2")` para nome/preço e para encontrar produtos COM estoque e SEM venda (capital parado).

### 4. Calcular indicadores
- **Cobertura (dias)** = saldo atual ÷ venda média diária
- **Crítico**: cobertura < 7 dias (ou saldo ≤ 0 com venda no período → ruptura ativa)
- **Atenção**: 7–15 dias
- **Produto morto**: saldo > 0 e zero vendas no período (capital parado = saldo × preço). **Só classificar como morto em cobertura total.** Em cobertura parcial, um produto pode não ter aparecido só porque o pedido dele ficou fora dos 150 detalhados — nesse modo o rótulo passa a ser "sem venda na amostra (não confirmado)", e não entra em recomendação de liquidação nem de descontinuação.
- **Sugestão de compra** = (venda diária × lead time informado pelo usuário, padrão 15 dias) + estoque de segurança (50%) − saldo atual. Arredondar para cima. Em cobertura parcial, a venda diária está **subestimada** (faltam pedidos), então a sugestão sai menor que a necessidade real: apresentar como piso, não como quantidade recomendada.
- Estoques **negativos** indicam erro de lançamento: destacar em seção própria de higiene de cadastro.

### 5. Relatório (formato de saída)
Abrir com a base da análise: período, pedidos no período, pedidos detalhados e modo (cobertura total ou parcial). Em modo parcial, o cabeçalho da tabela diz "Vendas na amostra" em vez de "Vendas 30d", para o número não ser lido como o total do período.

Tabela ordenada por criticidade:

| SKU | Produto | Vendas (período ou amostra) | Saldo | Cobertura | Status | Sugestão compra |

Seguido de: lista de saldos negativos, top 3 ações recomendadas e — **apenas em cobertura total** — a lista de produtos mortos com capital parado. Se o usuário pedir visual, gerar gráfico de barras de cobertura via widget.

### 6. Ação opcional
Se o usuário aprovar a reposição, localizar fornecedor com `listProductSuppliers` e criar o pedido com `createPurchaseOrder` — **somente com confirmação explícita do usuário, item a item**, pois é uma operação de escrita.

**Em cobertura parcial, não criar pedido de compra sem antes dizer ao usuário que a quantidade veio de demanda subestimada** e oferecer refazer a análise em janela menor com cobertura total. Comprar de menos por causa de amostra é caro de perceber: o efeito só aparece na ruptura seguinte. Se ele optar por seguir assim, registrar na confirmação que a base é parcial.

## Cuidados
- Use sempre o parâmetro `fields` para reduzir tokens.
- Ignore pedidos cancelados (verifique `situacao` via `listStatusModules` se necessário).
- Se a conta tiver poucos dados, informe isso em vez de extrapolar estatísticas.