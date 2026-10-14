---
name: analista-rentabilidade-real
description: "Analista de Rentabilidade Real via MCP Bling. Calcula a margem líquida de cada venda (preço − custo − frete − comissão − impostos − devoluções) por SKU, por canal ou consolidado. Identifica produtos que lucram e produtos que dão prejuízo de verdade. Use sempre que o usuário mencionar rentabilidade, margem, lucro, prejuízo, \"quanto ganho com este produto\", análise de lucratividade, ou pedir \"Análise de Rentabilidade\"."
---

# Analista de Rentabilidade Real

Responde: quanto EU GANHO de verdade com cada venda? (não apenas preço − custo da fábrica)

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listProducts getProduct"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listProducts` (custo em lote), `getProduct` (detalhe de um produto).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:analista-rentabilidade-real] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## O que entra na fórmula

Para cada item de venda em um pedido:

```
Margem Líquida = [Preço Unitário × Qtd] − [Custo Unitário × Qtd] − Frete Alocado − Comissão Marketplace − Impostos − Custo de Devoluções
```

Onde:
- **Preço unitário**: do pedido `getSalesOrder.itens[].valor`
- **Custo unitário**: `listProducts(codigos: [SKUs], fields: ["id","codigo","precoCusto"])` — uma chamada resolve o custo de vários SKUs. **Atenção ao caminho do campo:** `precoCusto` só existe na raiz na *listagem*; em `getProduct` (produto individual) o custo está em `fornecedor.precoCusto`, não na raiz. Ler da raiz do `getProduct` retorna vazio, o custo entra como 0 e a margem sai igual à receita. Se `precoCusto` estiver realmente vazio no cadastro, alertar e não calcular margem para esse SKU (ver Validação de dados).
  
  **O que esse custo é, e o que ele não é:** `precoCusto` é o custo do **fornecedor padrão cadastrado**, um valor de cadastro. Ele não é o **custo médio ponderado do estoque**, que se forma pelas movimentações reais (entradas por NF-e, com frete e impostos rateados) e é o número que a contabilidade usa. Os dois divergem quando o preço de compra muda, quando há mais de um fornecedor para o mesmo SKU, ou quando frete e DIFAL entram no custo da entrada. A API v3 não expõe o custo médio nem o histórico de movimentações, então a margem daqui é **a melhor aproximação disponível**, não o número contábil. Dizer isso ao usuário quando ele questionar divergência de custo, em vez de defender o valor.
- **Frete alocado**: `getSalesOrder.transporte.frete` ÷ (qtd total de itens) × (qtd deste item)
- **Comissão marketplace**: `getSalesOrder.taxas.taxaComissao` em %, com o canal identificado por `intermediador` (raiz do pedido, irmão de `loja`)
- **Impostos**: **não há alíquota real na API.** Perguntar ao usuário a alíquota efetiva do regime dele; se ele não souber, calcular a margem **antes de impostos** e dizer isso no relatório. Não assumir alíquota por categoria: chutar 18% para roupa ou 25% para eletrônico produz margem falsa com aparência de cálculo fiscal.
- **Custo de devoluções**: a API não expõe devolução (ver gestor-devolucoes-rma, onde o número é estimativa por heurística). Tratar como **entrada opcional do usuário**: se ele informar um percentual de provisão, aplicar e rotular; se não informar, deixar o termo fora da fórmula e registrar que a margem não considera devoluções. Nunca inventar percentual.

## Fluxo de execução

### 1. Definir período e filtros
Padrão: últimos 30 dias. Aceitar período customizado. Aceitar filtro por: canal (loja.id), categoria (produto), SKU específico.

### 2. Coletar vendas detalhadas

**Orçamento de chamadas.** A API permite 3 requisições por segundo, e esta skill precisa de `getSalesOrder` por pedido (é lá que estão itens, frete e comissão). **Teto de 150 chamadas de `getSalesOrder` por execução** (~50 s). Se a janela tiver mais pedidos que isso, amostrar os **150 de maior valor** — aqui o viés por valor é aceitável e desejável, porque margem é ponderada por receita e os pedidos maiores dominam o resultado. Informar ao usuário a cobertura obtida e oferecer recortar o período ou filtrar por canal/SKU para analisar 100% dos pedidos.

`listSalesOrders(dataInicial, dataFinal)` + `getSalesOrder` na amostra definida acima. Extrair:
- SKU / ID produto
- Preço de venda, quantidade
- Frete
- Intermediador (marketplace) e comissão em %
- Data (para análise de tendência)

### 3. Enriquecer com custo
Reunir os SKUs únicos da amostra (não por linha de pedido) e buscar o custo **em lote**:

```
listProducts(codigos: [SKU1, SKU2, ...], fields: ["id","codigo","precoCusto"])
```

Uma chamada resolve muitos SKUs, o que é bem mais barato que um `getProduct` por produto e mantém a skill dentro do limite de requisições. Se a lista de SKUs for longa, quebrar em blocos e paginar conforme `has_more`. Guardar o custo por SKU e reutilizar na sessão.

Se precisar do detalhe de um produto específico via `getProduct`, o custo está em **`fornecedor.precoCusto`** — não na raiz da resposta.

**SKU sem custo disponível** (campo vazio no cadastro, ou não retornado): não calcular margem para ele e **não usar 0**. Custo 0 faz a margem sair igual à receita e o produto aparecer como o mais lucrativo do catálogo. Esses SKUs vão para uma seção própria, "sem custo cadastrado — margem não calculável", e ficam fora da tabela de margem, do ranking, da seção "produtos no vermelho" e de qualquer recomendação de descontinuar.

### 3b. Impostos
Ver regra de Impostos em "O que entra na fórmula", acima.

### 4. Calcular por item / SKU / canal
Agrupar por:
- **Por SKU**: margem média, volume, contribuição total (margem × qtd)
- **Por canal**: qual marketplace lucra mais
- **Por período**: tendência de margem

### 5. Relatório (formato de saída)
Abrir declarando a base do cálculo: quantos pedidos foram analisados, de quantos no período, se o recorte foi por maior valor, e se a margem é antes ou depois de impostos.

1. **Resumo da amostra analisada**: faturamento, custo, margem líquida R$ e % — rotular como "da amostra" sempre que a cobertura não for total, porque com recorte pelos 150 maiores pedidos isso não é o faturamento do período
2. Tabela: SKU | Vendas | Preço Médio | Custo | Margem Unit. | Qty | Margem Total | %
3. **Produtos no vermelho**: margem ≤ 0 (vendendo com prejuízo) — apenas SKUs com custo cadastrado
4. **Ranking de rentabilidade**: top 10 SKUs por margem total (não unitária)
5. **Análise por canal**: qual marketplace rende mais margem (nem sempre é o de maior faturamento)
6. **Gráfico de tendência**: margem média diária/semanal (Artifacts widget)
7. **Sem custo cadastrado**: SKUs excluídos do cálculo por não terem `precoCusto` — com o aviso de que a margem real deles é desconhecida, não zero
8. Recomendações: aumentar preço dos SKUs de alto custo, focar volume nos de boa margem. Sugerir **descontinuar** um SKU só quando ele tiver custo cadastrado e volume relevante na amostra — descontinuar produto é decisão difícil de reverter e não deve sair de dado incompleto

### 6. Validação de dados
- SKUs sem `precoCusto`: listar na seção 7 e oferecer preencher em lote (via curador-catalogo)
- Fretes anormalmente altos: sinalizar para revisão (pode estar concentrando custo em 1 pedido grande)

## Cuidados
- Descontos inline (% desconto aplicado no pedido): já estão em `valor` do item, descontar do preço de venda
- Comissão: confirmar se é % ou valor fixo (API traz %; revisar se seu marketplace cobra taxa flat)
- Impostos: variam por estado, regime (simples, normal). Recomendar confirmação com contador para figura exata.
- Devoluções: não são capturadas automaticamente — usar histórico de devolução do **gestor-devolucoes-rma** se disponível