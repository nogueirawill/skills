---
name: concierge-cliente-360
description: "Concierge do Cliente 360º via MCP Bling. Consulta consolidada de atendimento: localiza o cliente, monta a ficha com histórico de compras, status do pedido atual, situação da NF-e, forma de envio/rastreamento e pendências financeiras, com resposta pronta para enviar ao cliente. Use sempre que o usuário mencionar \"cadê o pedido do cliente\", status de pedido de um cliente, histórico de compras, rastreamento, atendimento, SAC, reclamação de cliente, segunda via de nota, ou perguntar o que um cliente específico já comprou."
---

# Concierge do Cliente 360º

Uma consulta, uma ficha completa: pedido + nota + envio + financeiro do cliente, com texto de resposta pronto. Substitui abrir quatro telas do Bling durante o atendimento.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listContacts getContact listSalesOrders getSalesOrder getNfe listLogistics listAccountReceivables getAccountReceivableBoletos"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listContacts`, `getContact`, `listSalesOrders`, `getSalesOrder`, `getNfe`, `listLogistics`, `listAccountReceivables`, `getAccountReceivableBoletos`. Skill 100% de leitura.

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:concierge-cliente-360] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Localizar o cliente
`listContacts(pesquisa: <termo>, criterio: 1)` — o parâmetro `pesquisa` já cobre nome, CPF/CNPJ, fantasia, e-mail e código, então uma única chamada atende qualquer termo que o usuário informe. Passar `criterio: 1` (Todos) é obrigatório: o padrão da API é `3` (Últimos incluídos), que esconde clientes antigos. Se o termo for um documento e a busca não retornar, tentar `numeroDocumento` (CPF/CNPJ sem pontuação) como segunda tentativa. Se nada retornar, reportar "cliente não encontrado na conta Bling" sem inventar dados.

Com **mais de um resultado, nunca assumir**: listar até 5 (os mais recentes) e pedir ao usuário para escolher; acima de 5, informar o total encontrado e pedir um termo mais específico. `getContact` para a ficha completa (chamar no máximo uma vez por cliente).

### 2. Histórico e pedido em questão
`listSalesOrders(idContato: <id>)` para o histórico resumido (data, canal, valor, status) — esses campos vêm da própria listagem. Para **o pedido em foco**, `getSalesOrder`: itens, valores, transporte e `notaFiscal.id`.

**Teto de 5 `getSalesOrder` por atendimento**: o pedido em foco e, no máximo, os 4 pedidos recentes que o usuário pedir para detalhar. Detalhar 12 meses de histórico pedido a pedido não cabe no ritmo de um atendimento.

### 3. Situação fiscal e envio
Se `notaFiscal.id ≠ 0`, `getNfe`: situação na Sefaz, número, chave de acesso (para segunda via de DANFE).

**Rastreamento** vem de `getSalesOrder.transporte.volumes[].codigoRastreamento`, ou seja só existe para os pedidos que foram detalhados. `listLogistics` serve para identificar **qual integração de logística está configurada na conta** — ela não devolve código de rastreio nem vínculo com pedido. Nunca preencher código de rastreio por inferência: informar um código errado ao cliente final custa mais que dizer "vou verificar".

### 4. Pendências financeiras
`listAccountReceivables(idContato: <id>, situacoes: [1, 3])` — apenas Em aberto e Parcialmente recebido, que é o que representa pendência real. Sem o filtro de situação vêm também títulos já recebidos e cancelados, e a ficha mostraria como devendo um cliente em dia. Se precisar recortar por período, usar `tipoFiltroData: "V"` junto com as datas, porque o padrão da API filtra por emissão. Se houver boleto pendente, `getAccountReceivableBoletos` pelo idOrigem para obter linha digitável/link de reenvio. Útil antes de aprovar nova venda, liberar troca ou negociar.

### 5. Saída — ficha 360º
1. **Dados Cadastrais**: perfil do contato (nome, documento, e-mail, canal de contato)
2. **Pedido em foco**: timeline venda → NF-e (com situação na Sefaz e chave de acesso) → envio e rastreamento, com datas e onde está travado
3. **Histórico de Pedidos**: últimos 12 meses — data, canal, valor, status. Sem coluna de rastreamento, que exigiria detalhar cada pedido; se o usuário precisar do rastreio de um pedido antigo, detalhar aquele pedido especificamente
4. **Situação Financeira**: títulos em aberto, vencidos, boletos pendentes
5. **Análise de Comportamento**: frequência de compra, ticket médio, canais preferidos
6. **Ações Recomendadas**: próxima melhor ação para o atendimento + resposta pronta para o usuário copiar e enviar ao cliente (o envio é sempre do usuário, nunca da skill)

## Cuidados
- Dados pessoais do cliente aparecem apenas na resposta ao usuário — nunca em resumos agregados ou exemplos.
- Se o atendimento evoluir para troca/devolução, direcionar para o fluxo fiscal correto: devolução usa Natureza de Operação própria (indicador de devolução, CFOP de entrada), nunca a natureza de venda. A emissão em si segue o fluxo da esteira-faturamento-guiado com confirmação.
