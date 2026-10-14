---
name: gestor-devolucoes-rma
description: "Gestor de Devoluções e RMA via MCP Bling. Monitora devoluções por produto e canal, calcula taxa de devolução, identifica produtos problemáticos e mede impacto em reputação de marketplace. Use sempre que o usuário mencionar devoluções, trocas, RMA, taxa de devolução, produtos com qualidade ruim, reclamações de clientes, ou pedir \"Relatório de Devoluções\"."
---

# Gestor de Devoluções e RMA

Rastreia devoluções: que produto devolvem mais, em qual canal, qual o padrão, e o impacto na reputação.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listAccountReceivables listStatusModules listStatusModuleTransitions getContact listProducts"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listAccountReceivables` (títulos devolvidos), `listStatusModules` e `listStatusModuleTransitions` (discovery de situações), `getContact`, `listProducts`.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:gestor-devolucoes-rma] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

## Sinais de devolução na API

Uma devolução é capturada quando:
- Um pedido foi originalmente atendido (`situacao.id` = autorizado/completo)
- Mas depois recebeu uma **situação de devolução** ou há **múltiplos pedidos do mesmo cliente em data próxima** (correlação com devolução)
- Ou em `listAccountReceivables`, o campo `situacao = 4` (**Devolvido**)

Enum de contas a receber (fonte: resource `bling://guia/situacoes`): `1` Em aberto, `2` Recebido, `3` Parcialmente recebido, `4` Devolvido, `5` **Cancelado**. Atenção: `5` é Cancelado, **não** devolução parcial — incluí-lo no filtro de devoluções conta títulos cancelados como devolvidos e infla a taxa reportada. Devolução parcial, quando fizer sentido analisar, é `3` (Parcialmente recebido), e mesmo assim é indício, não confirmação.

**Importante**: A API do Bling NÃO tem campo explícito "devolvido" nos pedidos. Toda devolução aqui é **reconstruída por heurística**, não lida de um campo:
- Pedidos com `situacao.id` duplicado em curto período para mesmo cliente = correlação de devolução
- Títulos com `situacao = 4` em contas a receber (devoluções de pagamento)

Por isso **todo número de devolução produzido por esta skill é estimativa** e deve ser apresentado como tal, com a regra que o gerou explicitada. Nunca afirmar ao usuário que "houve N devoluções" como fato consolidado do sistema.

## Fluxo de execução

### 1. Mapear situações
Seguir o procedimento de discovery do `bling://guia/situacoes`: `listStatusModules` para conhecer os IDs reais de "autorizado", "cancelado", "devolvido" na conta, classificando pela heurística de nomes que o guia define. Uma vez por sessão. Contas a receber não precisam de discovery — o enum é fixo e está no próprio guia.

### 2. Coletar vendas do período (padrão: últimos 30 dias)
`listSalesOrders(dataInicial, dataFinal, fields: ["id","numero","data","total","contato","loja","situacao"])` com todos os status (não filtrar). Comparar com o mesmo período do mês anterior para tendência.

### 3. Inspecionar pedidos suspeitos de devolução
Para cada cliente (`contato.id`), se houver 2+ pedidos do mesmo SKU em < 5 dias de intervalo:
- Chamar `getSalesOrder` em ambos (**teto de 150 chamadas por execução** no total do passo; se houver mais pares candidatos, priorizar os de maior valor e declarar quantos pares foram verificados)
- Verificar se um tem situação de cancelamento/devolução
- Marcar como **potencial devolução**

Refinar com `listAccountReceivables(situacoes: [4], tipoFiltroData: "V", dataInicial: ..., dataFinal: ..., buscar_tudo: true)` para captar devoluções de pagamento (`4` = Devolvido). Não incluir `5`, que é Cancelado. Delimitar o mesmo período da análise: sem datas, a API assume o último ano e a taxa de devolução sai comparando um ano de títulos devolvidos com 30 dias de vendas. O `tipoFiltroData: "V"` é necessário porque o padrão filtra por emissão.

### 4. Calcular indicadores
- **Taxa de devolução %** = pedidos devolvidos ÷ pedidos totais do período
- **Taxa por SKU** = devoluções do SKU ÷ vendas do SKU (produtos com > 20% = crítico)
- **Taxa por canal** = qual marketplace tem mais devoluções (impacto em reputação)
- **Valor impactado**: soma dos valores devolvidos

O **motivo** da devolução não está disponível: o MCP não expõe comentários nem avaliações de produto, e o pedido não carrega justificativa de devolução. Não inferir motivo — quando o usuário perguntar "por quê", responder que o dado não existe na API e sugerir olhar o painel do marketplace.

### 5. Relatório (formato de saída)
1. **Resumo de devolução (estimado)**: taxa geral, comparação vs. período anterior, trending ↑/↓ — abrir o relatório com uma linha declarando o método: "devoluções estimadas por correlação de pedidos e títulos devolvidos; o Bling não expõe campo de devolução"
2. Tabela: SKU | Vendas período | Devoluções (est.) | Taxa % (est.) | Canal principal
3. Top 5 produtos com maior taxa (Red Flag)
4. Top 5 canais com maior taxa de devolução
5. Clientes com padrão de devolução recorrente (possível abuso)
6. Impacto estimado em reputação marketplace (marketplaces punem taxa > 5%)

### 6. Ação recomendada
- Produtos com taxa > 20%: revisar qualidade/descrição
- Canais com taxa alta: reforçar qualidade do envio (embalagem)
- Clientes seriais: considerar filtro de risco no checkout

## Cuidados
- Defeitos logísticos (produto chega quebrado) vs. insatisfação (produto não corresponde à descrição) precisam de contexto — a API por si não diferencia.
- Devoluções parciais (cliente devolveu 1 de 3 itens) exigem cálculo granular.