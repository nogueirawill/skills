---
name: sentinela-expedicao
description: "Sentinela de Expedição e Logística via MCP Bling. Monitora pedidos aprovados que ainda não foram faturados (sem NF-e) ou postados (sem código de rastreamento), mede o tempo venda→NF-e→postagem e aponta gargalos que ameaçam o SLA dos marketplaces. Use sempre que o usuário mencionar expedição, envio, SLA, atraso de postagem, pedidos parados, rastreamento, \"pedidos sem nota\", reputação no marketplace, ou pedir o \"Checkup de Expedição\"."
---

# Sentinela de Expedição e Logística

Garante o SLA de entrega monitorando o funil pedido → NF-e → postagem.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listSalesOrders getSalesOrder listStatusModules listStatusModuleTransitions listNfes listLogistics"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listSalesOrders`, `getSalesOrder`, `listStatusModules`, `listStatusModuleTransitions`, `listNfes`, `listLogistics`. **Skill 100% de leitura** — não emite, não autoriza e não altera nada.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:sentinela-expedicao] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Sinais de gargalo (aprendidos da estrutura real da API)

No retorno de `getSalesOrder`:
- `notaFiscal.id` = **0** → pedido **sem NF-e vinculada** (não faturado)
- `transporte.volumes[].codigoRastreamento` = **""** → **sem rastreio** (não postado)
- `data` vs. hoje → idade do pedido
- `situacao.id` → resolver o nome real seguindo o procedimento de discovery do `bling://guia/situacoes` (pedido de venda usa IDs dinâmicos por conta; os valores variam, então não fixar números no raciocínio). Fazer o discovery uma vez e reutilizar o mapa pelo resto da sessão. Esta é uma consulta operacional, então valem as regras de apresentação do guia para esse tipo — priorizar o que está em andamento.
- `intermediador.nomeUsuario` e `intermediador.cnpj` — identificam o canal de marketplace. Atenção: `intermediador` está na **raiz** do pedido, é irmão de `loja` (não `loja.intermediador`). Nomear o canal preferencialmente por `intermediador.nomeUsuario`, combinando com `loja.id` para separar as lojas configuradas na conta. Quando o prompt mencionar um marketplace específico, filtrar por esses campos; se `intermediador` vier vazio, tratar como venda direta/manual em vez de "canal não identificado"

## Fluxo de execução

### 1. Mapear situações da conta
Seguir o procedimento de discovery do `bling://guia/situacoes` para pedido de venda: `listStatusModules` → identificar o módulo de Vendas e os IDs de "Em aberto", "Em andamento", "Atendido", "Cancelado", classificando pela heurística de nomes que o guia define. Uma vez por sessão.

### 2. Buscar pedidos recentes não finalizados
```
listSalesOrders(dataInicial: hoje-7d, dataFinal: hoje,
                idsSituacoes: [ids não-cancelados],
                fields: ["id","numero","data","total","situacao","loja"])
```

### 3. Inspecionar cada pedido com mais de 24h
`getSalesOrder` para verificar `notaFiscal.id` e `codigoRastreamento`. Classificar:
- 🔴 **> 48h sem NF-e** — risco crítico de SLA
- 🟠 **24–48h sem NF-e ou com NF-e mas sem rastreio**
- 🟡 **< 24h pendente** — dentro do prazo, monitorar

Para prompts de análise de causa raiz (P4), chamar `getSalesOrder` em até 10 pedidos representativos para identificar o padrão de gargalo — a profundidade de análise tem prioridade sobre minimizar chamadas. Priorizar os pedidos mais antigos ou de maior valor quando a fila exceder 10 itens.

### 4. Cruzar com notas fiscais
`listNfes(tipo: "1", dataEmissaoInicial, dataEmissaoFinal, fields: [...])` para medir **tempo médio venda → emissão** e detectar NFs rejeitadas/pendentes que travam o envio. `listLogistics` para conferir as logísticas configuradas quando houver dúvida sobre o serviço de envio. Se `listLogistics` retornar HTTP 403, registrar "dados logísticos sem permissão de acesso" e continuar.

### 5. Relatório (formato de saída)
1. **Fila de gargalos**: Pedido | Canal | Data | Horas em espera | Falta NF? | Falta rastreio? | Valor
2. KPIs: tempo médio venda→NF, % de pedidos postados em < 24h, valor total represado na fila
3. Alertas por canal (marketplace com mais atrasos)
4. Ação sugerida pedido a pedido

### 6. Encaminhamento (esta skill não emite nota)
A sentinela diagnostica o gargalo e entrega a fila; a emissão fiscal é de outra skill. Ao fechar o relatório, indicar o próximo passo conforme a causa:

- **Pedidos sem NF-e** → **esteira-faturamento-guiado**, informando quais pedidos. Ela pré-valida estoque e cadastro fiscal antes de emitir, gera e autoriza a NF-e, baixa estoque e atualiza status, tudo com confirmação. Emitir sem essa pré-validação é o caminho mais curto para rejeição na Sefaz, e é por isso que a sentinela não faz esse passo.
- **NF-e emitida mas rejeitada ou pendente na Sefaz** → **vigia-fiscal-nfe**, que investiga o motivo da rejeição.
- **Cadastro fiscal incompleto travando a emissão** (NCM, GTIN, origem) → **preflight-fiscal-natureza** para gerar a fila de correção, e **curador-catalogo** para aplicar em lote.
- **Pedido sem estoque para faturar** → **saneador-estoque-lotes**.

Nunca gerar nem autorizar documento fiscal a partir desta skill, mesmo que o usuário peça: encaminhar para a esteira, onde o fluxo tem as validações e as confirmações certas.

## Cuidados
- Janela padrão de 7 dias; nunca varrer o histórico inteiro pedido a pedido.
- Pedidos de venda manual (sem `numeroLoja`) podem não exigir postagem — sinalizar separadamente.