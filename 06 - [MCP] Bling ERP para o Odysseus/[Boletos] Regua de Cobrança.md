---
name: regua-cobranca-boletos
description: "Régua de Cobrança com Boletos via MCP Bling. Age sobre a inadimplência: localiza títulos vencidos e a vencer, recupera os boletos (linha digitável/link) para reenvio, concilia com borderôs para não cobrar título já liquidado, prioriza a fila por valor × atraso e prepara minutas de mensagem de cobrança em três tons. Use sempre que o usuário mencionar cobrança, inadimplência, clientes devendo, boletos vencidos, reenviar boleto, régua de cobrança, títulos em atraso, ou perguntar \"quem está me devendo\"."
---

# Régua de Cobrança (Boletos e Borderô)

A saude-financeira-bling fotografa o caixa; esta skill age sobre a inadimplência: encontra, valida, prioriza e prepara a cobrança. O envio das mensagens é sempre do usuário.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listAccountReceivables getAccountReceivableBoletos getBordero getContact getNfe getSalesOrder"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listAccountReceivables`, `getAccountReceivableBoletos`, `getBordero`, `getContact`, `getNfe`, `getSalesOrder`. Skill 100% de leitura.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:regua-cobranca-boletos] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Levantar títulos
Duas faixas: vencidos (qualquer idade) e a vencer nos próximos 7 dias (cobrança preventiva). Agrupar por cliente.

```
listAccountReceivables(tipoFiltroData: "V", situacoes: [1, 3],
                       dataInicial: ..., dataFinal: ..., buscar_tudo: true)
```

Dois parâmetros são obrigatórios aqui e por motivos diferentes:

- **`tipoFiltroData: "V"`** — sem ele a API filtra por **data de emissão** (o padrão é `E`), não por vencimento. A consulta traria títulos *emitidos* na janela em vez de *vencendo* nela, ou seja a fila de cobrança sairia com o conjunto errado de títulos.
- **`situacoes: [1, 3]`** — apenas Em aberto e Parcialmente recebido. Sem esse filtro entram títulos já **recebidos** (`2`) e **cancelados** (`5`), e cobrar quem já pagou é justamente o erro mais caro desta skill. Para título `3` (Parcialmente recebido), cobrar somente o saldo remanescente, nunca o valor cheio.

Esse filtro protege contra o caso comum. A conciliação de borderô do passo 2 cobre o caso específico de baixa em lote bancário, que não altera a situação do título.

### 2. Conciliar antes de cobrar (obrigatório)
Cobrar título já pago é o erro mais caro da cobrança — esta etapa nunca pode ser pulada, mas o caminho depende do que está disponível:

- **Quando o usuário informar o número/ID de um borderô** (ou quando o título indicar liquidação em lote bancário), chamar `getBordero(idBordero)` e conferir os títulos ali baixados. Os liquidados saem da fila e vão para a seção "Boletos Já Quitados Excluídos".
- **Sem ID de borderô disponível**, não há como listar borderôs pela API: nesse caso **não afirmar que o título está quitado nem que está aberto**. Todo título com indício de liquidação vai para a Quarentena com a marcação "verificar baixa", e a Fila Priorizada informa que a conciliação de borderô não pôde ser feita nesta execução.

### 3. Validar o lastro fiscal
Para os **30 maiores títulos** da fila (teto de chamadas por execução), verificar via `getSalesOrder`/`getNfe` se a venda tem NF-e autorizada vinculada. Título de venda com nota cancelada/denegada entra em quarentena: cobrar sem lastro fiscal gera passivo de relacionamento e fiscal. Título não verificado entra na fila normalmente, mas sem a marca de lastro fiscal confirmado.

### 4. Enriquecer o devedor
`getContact` dos clientes da fila: canal de contato e histórico. Cliente recorrente e bom pagador com 1º atraso recebe tom diferente de reincidente.

### 5. Boletos para reenvio
`getAccountReceivableBoletos` pelo idOrigem de cada título confirmado → linha digitável e link, prontos para anexar na mensagem.

### 6. Saída — fila de cobrança
1. **Fila priorizada**: cliente, títulos, valor total, dias de atraso, score de prioridade (valor × atraso × reincidência)
2. **Quarentena**: títulos SEM lastro fiscal (venda com nota cancelada/denegada) ou com indício de liquidação sem confirmação (verificar baixa), com as ações recomendadas
3. **Minutas em três tons** por cliente: lembrete cordial (a vencer/atraso curto), cobrança formal (atraso médio), última notificação (atraso longo) — cada uma com os dados do boleto. A skill prepara; quem envia é o usuário.
4. **Indicadores**: total vencido, % da carteira inadimplente, aging (0–15 / 16–30 / 31–60 / 60+ dias)
5. **Boletos Já Quitados Excluídos**: títulos identificados no borderô como já liquidados — excluídos da fila de cobrança. Se um borderô foi consultado e nada foi encontrado: "nenhum boleto quitado identificado". Se nenhum borderô foi consultado (sem ID disponível): "conciliação de borderô não realizada nesta execução — confirmar baixas antes de enviar as cobranças".

## Cuidados
Nunca sugerir juros, multa ou ameaça de negativação sem o usuário informar a política de cobrança vigente. Dados do devedor aparecem apenas na resposta ao usuário.
