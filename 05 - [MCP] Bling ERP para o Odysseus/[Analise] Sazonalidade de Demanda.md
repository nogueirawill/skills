---
name: analista-sazonalidade-demanda
description: "Analista de Sazonalidade e Demanda via MCP Bling. Identifica períodos de alta/baixa demanda, sazonalidade por produto, tendências de crescimento/declínio. Prevê demanda para os próximos 30 dias baseado em padrão histórico e sazonalidade. Use sempre que o usuário mencionar sazonalidade, previsão de demanda, padrão de venda, produtos que caem/crescem, \"quantas unidades vou vender mês que vem\", ou pedir \"Análise de Sazonalidade\"."
---

# Analista de Sazonalidade e Demanda

Identifica padrões cíclicos e projeta demanda futura: quanto você venderá nos próximos 30 dias.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listProducts"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listProducts`.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:analista-sazonalidade-demanda] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Conceitos

### Sazonalidade
Variação previsível da demanda em períodos (mês, semana, dia). Ex.: Natal, volta às aulas, fim de mês aumentam vendas de certos produtos.

### Tendência
Crescimento ou declínio de longo prazo de um produto (curva ascendente = produto em alta; curva descendente = produto em declínio).

### Classificação de SKUs
- **Mover (crescimento)**: tendência positiva > 10% por trimestre
- **Shaker (declínio)**: tendência negativa > 10% por trimestre
- **Estável**: variação ≤ ±10% por trimestre

### Previsão Simples
Multiplicar a média histórica pela sazonalidade observada = demanda estimada.

```
Previsão = (Média histórica) × (Fator sazonal) + (Tendência)
```

## Fluxo de execução

### 1. Coletar histórico (janela padrão: últimos 12 meses)

**Orçamento de chamadas.** A API permite 3 requisições por segundo, então `getSalesOrder` pedido a pedido sobre 12 meses é inviável: 3.000 pedidos levariam cerca de 17 minutos e a análise não conclui. Trabalhar em duas camadas:

1. **Agregados (barato):** `listSalesOrders(dataInicial: hoje-365d, dataFinal: hoje, buscar_tudo: true, fields: ["id","numero","data","total"])`. Volume e receita por dia, semana e mês saem daqui — `data` e `total` bastam para toda a análise de sazonalidade e tendência agregada. **Não chamar `getSalesOrder` para isso.** Conferir `_meta.buscaCompleta`: se vier `false`, a varredura parou no meio e a análise é do intervalo efetivamente coberto, não dos 12 meses. Nesse caso, ou reduzir a janela (ex.: 6 meses) e refazer, ou seguir declarando a cobertura obtida — nunca apresentar como se fosse o ano inteiro.
2. **Nível de SKU (caro, sempre amostral):** `getSalesOrder` apenas quando a análise precisar descer a produto. **Teto de 150 chamadas por execução.** Amostrar de forma **estratificada por mês** — até ~12 pedidos por mês, distribuídos ao longo do período — e nunca por maior valor: amostra enviesada por valor distorce exatamente o padrão sazonal que se quer medir.

Reutilizar na sessão o que já foi obtido; não repetir a coleta se o usuário refinar a pergunta sobre o mesmo período.

Sempre declarar no relatório a cobertura real (ex.: "agregados sobre 2.847 pedidos em 12 meses; análise por SKU sobre amostra de 144 pedidos").

Agregar (a partir dos agregados da camada 1):
- Volume diário e receita diária por data — base de toda a sazonalidade e tendência

A quebra **por SKU** só existe na camada 2, ou seja sobre a amostra. Toda métrica por SKU (classificação Mover/Shaker, previsão) vale apenas para os SKUs presentes na amostra e deve dizer quantos pedidos a sustentam. Não extrapolar para SKUs que não apareceram na amostra.

### 2. Calcular sazonalidade por período

#### Semanal (dia da semana)
```
volume_segunda = MÉDIA(vendas todas as 2as-feiras últimos 12m)
volume_terça = MÉDIA(vendas todas as 3as-feiras)
... etc
fator_sazonal_segunda = volume_segunda / (MÉDIA de todos os dias)
```

Resultado: qual dia da semana vende mais (ex.: sexta vende 1,5x a média)?

#### Mensal (semana do mês, semana de calendário)
```
semana_1 = MÉDIA(vendas semanas 1-7 de cada mês)
semana_2 = MÉDIA(vendas semanas 8-14)
... semana_4
```

Resultado: começo/meio/fim de mês? Qual semana é mais fraca?

#### Por Mês (sazonalidade anual)
```
janeiro = MÉDIA(vendas todos os janeiros últimos 12m)
... dezembro
fator_sazonal_janeiro = janeiro / (MÉDIA todos os meses)
```

Resultado: quais meses são picos (Natal, Black Friday)?

### 3. Calcular tendência (crescimento/declínio)
Dividir os 12 meses em 4 trimestres:
```
T1 = SUM(jan+fev+mar) / 3
T2 = SUM(abr+mai+jun) / 3
T3 = SUM(jul+ago+set) / 3
T4 = SUM(out+nov+dez) / 3
```

Regressão linear simples:
```
tendência_mensal = (T4 − T1) / 12  em % ou unidades
```

Se positivo: produto cresce. Se negativo: declina.

### 4. Identificar "Movers" e "Shakers"
Classificar cada SKU como Mover, Shaker ou Estável conforme os limiares definidos em **Conceitos → Classificação de SKUs**. Esta classificação deve ser incluída sempre que os dados permitirem — em prompts mínimos, proceder autonomamente sem pedir clarificação.

### 5. Prever demanda próximos 30 dias

**Só prever SKU com base suficiente**: mínimo de 20 pedidos do SKU na amostra. Abaixo disso, listar o SKU como "amostra insuficiente para previsão" em vez de emitir número — previsão por SKU sobre 2 ou 3 pedidos comunica precisão que o dado não tem, e alimenta decisão de compra real.

Para cada SKU que atender ao mínimo:

**Base**: MÉDIA(vendas últimos 30 dias antes da previsão)

**Ajuste sazonal**: fator sazonal do mês que vem

**Ajuste tendência**: tendência × (dias até o fim do período previsão)

```
Previsão_30d = BASE × fator_sazonal_mês × (1 + tendência_mensal)
```

Faixa otimista/pessimista: derivar da **dispersão observada** do próprio SKU (ex.: desvio padrão do volume mensal no período), não de um percentual fixo. Faixa fixa de ±10% comunica uma precisão que não vem dos dados. Quando não houver observações suficientes para calcular dispersão, apresentar só a estimativa central e dizer que a faixa não é calculável.

### 6. Relatório (formato de saída)
1. **Resumo geral**: vendas médias/mês, sazonalidade mais forte, trending agregado
2. **Gráfico de sazonalidade semanal**: qual dia vende mais
3. **Gráfico de sazonalidade anual**: quais meses são picos
4. **Movers e Shakers**: SKUs em crescimento (oportunidade) e em declínio (risco)
5. **Previsão próximos 30 dias**: por SKU
   - Estimativa central
   - Range otimista/pessimista
   - Comparação com período anterior
6. **Insights por categoria**: produtos relacionados com padrão similar
7. **Recomendações**:
   - SKUs para reordenar já (vão crescer)
   - SKUs para não reordenar (vão cair)
   - Oportunidade de promoção em períodos fracos
   - Preparação de estoque para picos identificados

### 7. Ações opcionais
Se integrado com **radar-ruptura-estoque**, oferecer sugestão automática de recompra baseada em previsão de demanda.

## Cuidados
- **Dados insuficientes**: se histórico < 3 meses, indicar a limitação explicitamente e fornecer a análise parcial possível (tendência e classificação para o período disponível) — nunca recusar análise por dados escassos.
- **Anomalias históricas**: picos/quedas únicas (ex.: promoção massiva, campanha viral) inflacionam o fator sazonal. Considerar excluir outliers.
- **Novos produtos**: sem histórico não há sazonalidade nem previsão. Dizer isso claramente em vez de inventar um número. O caminho útil é comparar com um SKU análogo da mesma categoria que tenha histórico, deixando explícito que é proxy ("baseado no comportamento do SKU X, que não é o mesmo produto"). Nunca apresentar percentual de crescimento genérico como previsão.
- **Black Friday, Natal, etc.**: identificar automaticamente datas de pico historicamente e sinalizar se próximo período vai ter uma
- **Marketplace vs. loja própria**: sazonalidade pode divergir por canal — fazer análise por `loja.id` se tiver diversidade