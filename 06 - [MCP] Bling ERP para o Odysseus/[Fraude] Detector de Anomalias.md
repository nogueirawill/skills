---
name: detector-anomalias-fraude
description: "Detector de Anomalias e Fraude via MCP Bling. Identifica padrões suspeitos em vendas — picos anormais, quedas bruscas, múltiplas compras do mesmo cliente, devoluções altas, endereços duplicados, variações de preço estranhas. Use sempre que o usuário mencionar fraude, padrão suspeito, pico de vendas estranho, queda abrupta de vendas, múltiplas compras mesmo cliente, ou pedir \"Análise de Anomalias\"."
---

# Detector de Anomalias e Fraude

Encontra o "estranho" nos dados: picos inexplicados, quedas suspeitas, padrões de fraude.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder getContact listStatusModules"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `getContact`, `listStatusModules` (resolver IDs de situação).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:detector-anomalias-fraude] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Os IDs de situação de pedido de venda são customizáveis por conta — nunca chutar o ID de "cancelado". Consultar o resource `bling://guia/situacoes` para o mapa de situações e as regras de apresentação por tipo de consulta; se o módulo não estiver coberto lá, resolver com `listStatusModules` na primeira execução e reutilizar o mapa pelo resto da sessão.

## Padrões de anomalia detectados

### A. Picos e Quedas de Volume
- **Pico anormalmente alto**: vendas do dia > média 30d + 3 desvios padrão
- **Queda abrupta**: vendas do dia < média 30d − 2 desvios padrão (por mais de 3 dias seguidos)
- Comparar por canal para isolar: se só 1 marketplace caiu, pode ser algo específico daquela integração

### B. Fraude por Padrão de Cliente
- **Múltiplas compras rápidas**: 3+ pedidos do mesmo `contato.id` em < 4 horas (possível teste/roubo de cartão)
- **Compras em endereços duplicados**: 10+ clientes diferentes enviando para mesmo CEP/endereço (revenda ou fraude)
- **Valores incomuns**: cliente que sempre compra R$ 50 de repente compra R$ 5.000 (roubo?)

### C. Anomalias de Produto
- **Pico de vendas de 1 SKU**: produto X vendeu 100 un hoje vs. 2–3 un/dia normalmente (pode ser estoque malformado, duplicado, ou boom legítimo)
- **Devolução** não é medida aqui: o Bling não expõe campo de devolução no pedido, e esta skill não consulta contas a receber. Se o usuário perguntar sobre devoluções, encaminhar para **gestor-devolucoes-rma**, onde o número é reconstruído por heurística e apresentado como estimativa.
- **Preço anormalmente baixo**: item vendido a 10% do preço normal (erro de cadastro ou promoção não autorizada)

### D. Anomalias Operacionais
- **NF-e não emitida há dias**: pedido aprovado há 5+ dias sem nota (gargalo fiscal)
- **Sem rastreio há 3+ dias**: pedido aprovado não postado (SLA em risco)
- **Cancelamentos em massa**: 20%+ dos pedidos do dia foram cancelados (problema de integração?)

## Fluxo de execução

### 1. Definir baseline (30 dias anteriores ao período)
`listSalesOrders(dataInicial: hoje-60d, dataFinal: hoje-30d, buscar_tudo: true)` → calcular:
- Média diária de vendas (geral e por canal)
- Desvio padrão
- Top 5 produtos por volume
- Padrão normal de preço por SKU

O output deve referenciar explicitamente os dois janelas com suas datas (ex.: "baseline: 01/05–31/05; período analisado: 01/06–30/06") e o método estatístico aplicado (z-score, desvio padrão).

### 2. Coletar período em análise (últimos 30 dias)
`listSalesOrders(dataInicial: hoje-30d, dataFinal: hoje, buscar_tudo: true)`.

As duas listagens (baseline e período) usam `buscar_tudo` porque a comparação estatística exige o volume real dos dois lados: média e desvio padrão calculados sobre uma página de cada lado produzem z-score sem sentido.

**Detalhe por pedido tem teto.** As detecções que precisam de `getSalesOrder` (endereço de entrega, itens, preço praticado) custam uma requisição por pedido, a 3 por segundo: **teto de 150 `getSalesOrder` por execução**, priorizando os pedidos que já saíram como suspeitos nas detecções agregadas (pico de volume, cliente com múltiplas compras, valor fora do padrão). Não varrer o período inteiro pedido a pedido.

Consequência a declarar no relatório: detecção de **endereços duplicados** só é conclusiva sobre os pedidos efetivamente inspecionados. Com cobertura parcial, informar quantos pedidos foram lidos e não afirmar que o padrão não existe fora da amostra — ausência de evidência aqui não é evidência de ausência.

### 3. Executar detecções

#### A. Volume (Picos/Quedas)
Para cada dia D da janela:
```
z_score = (vendas_dia_D − media_30d) / desvio_padrão_30d
if z_score > +3: PICO ANÔMALO
if z_score < −2 por 3+ dias: QUEDA ANÔMALA
```

#### B. Fraude por Cliente
Para cada `contato.id`:
- Contar pedidos últimas 24h: se > 5, FLAG
- Contar pedidos últimas 4h: se > 2, RED FLAG (teste de cartão roubado)
- Valor médio do cliente vs. último pedido: se Δ > 300%, FLAG
- Clientes com volume em um único dia excedendo **3× a média baseline** entram na lista de clientes a revisar, sempre com a métrica exata que disparou o alerta ao lado do nome (ex.: "R$ 4.200 em 12/08, média diária da loja R$ 1.300"). Comprar 3× a média num dia é comportamento comum de atacadista, empresa ou compra de kit, então este é um **indício estatístico, não confirmação de fraude** — o texto da seção deve dizer isso, e a decisão é sempre do usuário com verificação manual.

#### C. Endereços Duplicados
Extrair `etiqueta` (endereço de entrega) de `getSalesOrder.transporte` para cada pedido. Contar clientes por endereço; **10 ou mais** clientes diferentes no mesmo CEP/rua, FLAG (possível fraude em massa).

#### D. Anomalias por SKU
Para cada produto vendido:
- Volume hoje vs. média 30d: se > 10x, FLAG (especialmente se novo produto)
- Preço hoje vs. histórico: se < 50% do usual, FLAG

#### E. Atrasos Operacionais
Dos pedidos dos últimos 7 dias:
- `notaFiscal.id = 0` há > 5 dias: FLAG
- Sem rastreio há > 3 dias: FLAG

### 4. Compilar scores
Cada anomalia recebe **peso** (1 = baixo, 2 = médio, 3 = crítico). Somar por pedido/cliente/canal. Pedidos com score ≥ 5 = review manual.

### 5. Relatório (formato de saída)
1. **Sumário de alertas**: N anomalias detectadas, N críticas, tendência vs. período anterior
2. **Tabela de anomalias**: tipo | data | pedido/SKU/cliente | score | descrição | ação sugerida
3. **Gráfico de volume**: linha do faturamento diário com pontos destacados de pico/queda
4. **Clientes a revisar** (até 10, por score): abrir a seção declarando que são indícios estatísticos que exigem verificação manual, não fraude confirmada. Para cada um, o critério que disparou (score ≥ 5, mais de 3× a média diária, 3+ pedidos em 4h) e a métrica com a base de comparação
5. **Produtos em alerta**: SKUs com picos ou quedas anormais
6. **Operacional**: pedidos travados, SLA em risco
7. **Recomendações**: sempre de verificação, nunca de punição automática — "revisar manualmente os pedidos com score ≥ 5", "conferir se o endereço X é de um cliente atacadista antes de qualquer ação", "checar qualidade do SKU Y"

### 6. Encaminhamento (esta skill é 100% de leitura)
O detector aponta e explica; nenhuma ação sobre cliente ou pedido sai daqui. Bloquear cliente e cancelar pedido não são operações disponíveis nesta skill, e prometer isso ao usuário para depois não cumprir é pior que não oferecer.

Ao fechar, encaminhar conforme o achado:

- **Pedido suspeito que o usuário decidiu não atender** → alterar situação é operação da própria tela do Bling, ou da esteira quando o fluxo for de faturamento
- **Cliente com padrão recorrente de devolução** → **gestor-devolucoes-rma** para medir a taxa dele
- **Pedidos travados sem NF-e ou sem rastreio** → **sentinela-expedicao**
- **Preço fora do padrão por erro de cadastro** → **curador-catalogo**

## Cuidados
- **Falso positivo**: picos legítimos (Black Friday, promoção autorizada, influencer mencionou seu produto) não são fraude. Contexto importa.
- **Sazonalidade legítima**: fim de mês, volta às aulas, Natal — geram picos reais, não anormais. Considerar calendário.
- **Novas integrações**: quando conecta novo marketplace, primeiro mês terá padrão diferente — não é anomalia.
- **Indisponibilidade de ferramenta**: se uma chamada MCP retornar erro de servidor (503/504), registrar "ferramenta temporariamente indisponível" para aquele dado e continuar a análise com os dados obtidos — nunca abortar a sessão por falha parcial.
- Machine learning simplificado: este detector usa estatística descritiva (z-score, média, desvio). Para fraude real, sistemas bancários usam ML mais robusto.