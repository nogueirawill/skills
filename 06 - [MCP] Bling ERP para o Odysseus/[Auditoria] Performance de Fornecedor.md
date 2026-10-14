---
name: auditor-fornecedores-performance
description: "Auditor de Fornecedores (Performance) via MCP Bling. Mede prazo de entrega real (NF-e de entrada vs. data prevista), conformidade de quantidade recebida, preço médio e tendência de custo por fornecedor, com score e ranking. Use sempre que o usuário mencionar fornecedores, atrasos de entrega, prazo de fornecimento, conformidade de pedido de compra, aumento de custo, problema com fornecedor, ranking de suppliers, ou pedir \"Auditoria de Fornecedores\"."
---

# Auditor de Fornecedores (Performance)

Avalia cada fornecedor: prazo, qualidade, conformidade, trending de preço e custo-benefício.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listPurchaseOrders getPurchaseOrder getContact listNfes"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listPurchaseOrders`, `getPurchaseOrder`, `getContact`, `listNfes`.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:auditor-fornecedores-performance] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Sinais de performance

### A. Prazo de Entrega
- **Esperado**: data de pedido vs. `dataPrevista` (lead time negociado)
- **Realizado**: data de pedido vs. data da NF-e de entrada (quando chegou de verdade)
- **Delta**: atraso em dias = data realizada − dataPrevista
- **Fonte de dados de entrega**: a data de emissão da NF-e de entrada (via `listNfes tipo: "0"`) é a evidência de recebimento. **Pedido sem NF-e de entrada fica fora do cálculo de prazo** e vai para uma coluna própria, "sem evidência de recebimento". Não usar `dataPrevista` como se fosse a data realizada: isso produz atraso zero, ou seja o pedido entra como entregue no prazo, e o fornecedor que não emite NF-e de entrada aparece com prazo perfeito, sobe no ranking e pode receber a recomendação de priorizar. É valor fabricado alimentando o maior peso do score (prazo — ver fórmula no passo 4).
- Quando os pedidos sem evidência passarem de 30% do total de um fornecedor, o prazo dele não é mensurável: marcar como "prazo não apurado" e não emitir alerta de severidade por atraso.
- Média histórica: atraso médio dos últimos 12 meses, calculado só sobre pedidos com NF-e de entrada

### B. Conformidade de Quantidade
- **Solicitado**: qtd do pedido de compra
- **Recebido**: qtd da NF-e de entrada
- **Falta**: se recebeu menos, quanto faltou (quebra de promessa = custo de ruptura)

### C. Qualidade (Proxy via Devoluções)
- Produtos recebidos deste fornecedor que depois foram devolvidos por cliente
- Taxa de defeito = (unidades com defeito / unidades recebidas) — estimar via devolução
- Atrasos combinados com defeito = fornecedor pior que o esperado

### D. Preço e Tendência
- Preço médio pago nas últimas 12 compras
- Trending (subindo? caindo? estável?)
- Comparar preço atual com preço médio: % variação

### E. Conformidade Fiscal
- NF-es emitidas em tempo correto (entrada registrada sem atraso)
- Rejeitadas/denegadas (problema fiscal com fornecedor)

## Fluxo de execução

### 1. Mapear fornecedores
`listPurchaseOrders(dataInicial: hoje-365d, dataFinal: hoje)` → extrair `fornecedor.id` único. `getContact()` para resolver o nome, uma vez por fornecedor (nunca por pedido) e no máximo 50 por execução — acima disso, priorizar os fornecedores com mais compras no período e exibir os demais pelo `fornecedor.id`.

### 2. Coletar histórico (janela padrão: últimos 12 meses)

**Orçamento de chamadas.** A API permite 3 requisições por segundo, então buscar todos os pedidos de compra de 12 meses um a um pode não concluir. **O teto de 150 `getPurchaseOrder` por execução é rígido** e tem precedência sobre qualquer outra regra de amostragem.

Como distribuir os 150, nesta ordem:

1. Ordenar os fornecedores por volume de compra no período.
2. Alocar até 5 pedidos por fornecedor, começando pelos de maior volume, **até o teto acabar**. Com 50 fornecedores, cobrir 5 de cada exigiria 250 chamadas, ou seja não cabe: os fornecedores que não receberem alocação ficam de fora e são listados como "não avaliados nesta execução", não como fornecedores ruins.
3. Se sobrar orçamento depois de cobrir todos com 5, distribuir o restante nos de maior volume para dar mais base ao score deles.

Fornecedor que ficar com menos de 5 pedidos amostrados entra como "base insuficiente": aparece no ranking, mas com score indicativo e sem alerta de severidade. Reutilizar na sessão o que já foi coletado.

Quando houver mais fornecedores que o orçamento cobre, dizer isso na abertura do relatório e sugerir recortar por fornecedor específico ou por janela menor, que devolve avaliação completa de um subconjunto.

Para cada fornecedor:
- `getPurchaseOrder` na amostra definida acima
- Para cada pedido: data, `itens[].valor` unitário, `itens[].quantidade`, status
- Corresponder com `listNfes(tipo: "0")` para captar data real de entrada (NF-e emitida)

### 3. Calcular indicadores

#### Prazo
```
atraso_médio = MÉDIA(data_NF − dataPrevista) em dias
percentual_no_prazo = (pedidos com atraso ≤ 0 / total) × 100%
maior_atraso = MAX(atraso)
```

#### Quantidade
```
conformidade_qtd = (pedidos com qtd recebida = qtd solicitada / total) × 100%
falta_total = SUM(qtd_solicitada − qtd_recebida para falta > 0)
```

#### Preço
```
preço_médio = MÉDIA(preço_unitário) dos últimos 12 meses
trending = comparar preço últimos 3 meses vs. primeiros 3 meses
variação_atual = (preço_hoje − preço_médio) / preço_médio
```

#### Qualidade — fora do score, por falta de fonte
Não há dado de qualidade por fornecedor na API: o Bling não expõe devolução no pedido, e ligar um produto devolvido ao fornecedor que o vendeu depende de duas inferências encadeadas (a heurística de devolução, que já é estimativa, mais o vínculo produto→fornecedor). Somar isso a um score que o usuário lê como nota de fornecedor produziria número frágil com aparência de medição.

Por isso a qualidade **não entra no score**. Quando o usuário tiver dados próprios de defeito ou devolução por fornecedor, ele pode informar e a skill apresenta em seção separada, rotulada como informação do usuário. Se ele quiser a estimativa por devoluções, encaminhar para **gestor-devolucoes-rma**, que expõe a heurística e seus limites.

### 4. Calcular Score Geral (0–100)
Somente sobre dado medido:
```
score = (
  prazo_no_prazo × 0,40 +          // 40% prazo (NF-e de entrada vs. dataPrevista)
  conformidade_qtd × 0,35 +        // 35% quantidade (qtd recebida vs. solicitada)
  (100 − |variação_preço%|) × 0,25 // 25% estabilidade de preço
)
```

Classificação: A (80+), B (60-79), C (40-59), D (<40).

O score exige base mínima. Fornecedor com menos de 5 pedidos na amostra, ou com mais de 30% dos pedidos sem NF-e de entrada, recebe "score indicativo" e **nenhum alerta de severidade** — um único atraso não define um fornecedor.

### 5. Relatório (formato de saída)
1. **Ranking de fornecedores**: Fornecedor | Score | Prazo Médio | Pedidos c/ NF-e (base) | Conformidade % | Preço Médio | Trending. A coluna de base deixa visível de quantos pedidos cada linha saiu, o que é o que separa "fornecedor ruim" de "pouca informação"
2. **Detalhes por fornecedor**: (se solicitado)
   - Últimas 10 compras, data chegada vs. prevista
   - Maior atraso e data
   - Histórico de falta de quantidade
   - Gráfico de preço (trending)
3. **Alertas por fornecedor** (severidade obrigatória, só para fornecedor com base suficiente):
   - 🔴 **Crítico**: atraso médio > 20 dias OU conformidade de quantidade < 70% — nomear fornecedor e a métrica que disparou, com a base ao lado (ex.: "atraso médio de 24 dias em 8 pedidos com NF-e de entrada")
   - 🟠 **Atenção**: atraso médio 10–20 dias OU conformidade QTD < 90% OU preço subindo > 10% em 3 meses
   - 🟡 **Info**: atraso < 10 dias OU variação de preço menor
   - **Sem alerta**: fornecedor com menos de 5 pedidos na amostra, ou com mais de 30% dos pedidos sem NF-e de entrada. Aparece no ranking com "base insuficiente" no lugar da severidade
   - Apresentar tabela de ranking ANTES dos blocos narrativos por fornecedor
4. **Recomendações**:
   - Fornecedores a priorizar (Score A com base suficiente)
   - Fornecedores para revisão/negociação (Score C/D com base suficiente)
   - SKUs com fornecedor único em risco de prazo (concentração de fornecimento)
   - Oportunidade de renegociar preço (queda de preço do mercado)

## Cuidados
- Atrasos legítimos (força maior, greve de transportadora) não são culpa do fornecedor
- Confundir atraso de NF-e com atraso de entrega física — a NF-e de entrada é a melhor evidência disponível, mas é a data do documento, não a da chegada física
- Fornecedor com base insuficiente (< 5 pedidos ou muitos sem NF-e) tem score indicativo e nenhum alerta
- Qualidade não é medida por esta skill: não há dado de defeito por fornecedor na API. Nunca afirmar que um fornecedor entrega produto com defeito a partir de inferência — é acusação com efeito comercial real sobre dado que o Bling não tem.