---
name: curador-catalogo
description: "Curador de Catálogo via MCP Bling. Executa a limpeza do catálogo: corrige cadastros em massa via patch (NCM, GTIN, descrição, dimensões, preço), organiza produtos em categorias e grupos, padroniza variações e cadastra produtos novos — sempre com plano de mudança aprovado antes. Use sempre que o usuário mencionar corrigir cadastro de produtos, arrumar o catálogo, categorizar produtos, criar categoria ou grupo, cadastrar produto novo, atualizar produtos em massa, padronizar variações, ou pedir uma \"faxina no catálogo\"."
---

# Curador de Catálogo

O otimizador-anuncios-sku diagnostica um SKU por vez; esta skill **corrige** o catálogo em escala: patches em massa, categorização e cadastro de novos produtos — com plano de mudança aprovado antes de qualquer escrita. Use curador-catalogo para correções em lote em múltiplos produtos; use otimizador-anuncios-sku para diagnosticar e otimizar um único SKU.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listProducts getProduct listProductVariations listProductCategories listProductGroups patchProduct createProduct createProductCategory createProductGroup"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listProducts`, `getProduct`, `listProductVariations`, `listProductCategories`, `listProductGroups`, `patchProduct`, `createProduct`, `createProductCategory`, `createProductGroup` (escritas — só com plano aprovado).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:curador-catalogo] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

Esta skill **não referencia `bling://guia/situacoes`**, ao contrário das demais 17: o fluxo toca só produtos, variações, categorias e grupos, e o único enum aqui é `criterio` (ativo/inativo/excluído em `listProducts`), que é filtro de cadastro, não situação de um registro. O guia não tem seção de produtos. Se este fluxo passar a usar `listStatusModules` ou qualquer entidade com situação, adicionar a referência então.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

## Fluxo de execução

### 1. Diagnóstico do catálogo
**Comportamento autônomo**: em prompts mínimos ou ambíguos ("curadoria do catálogo", "faxina no catálogo"), iniciar imediatamente com `listProducts` — sem pedir clarificação antes das primeiras chamadas de ferramenta. `listProducts(tipo: "P", criterio: 2, buscar_tudo: true)` (produtos, excluindo serviços; `criterio=2` = Ativos) + `getProduct` **somente quando a listagem não bastar** (a listagem já traz nome, código, preço, custo, estoque e situação, o que cobre a maior parte do diagnóstico). Teto de 100 `getProduct` por execução, priorizando o lote que o usuário quer corrigir. O `buscar_tudo` é necessário porque o diagnóstico reporta contagem e percentual por categoria, o que exige o universo e não uma página. Se `_meta.buscaCompleta` vier `false`, os percentuais são da parcela varrida: dizer isso explicitamente ("análise parcial: N de M produtos") e oferecer recortar por categoria para fechar o catálogo em partes. Passar `criterio` explicitamente importa: o padrão da API é `1` (Últimos incluídos), que não é o universo de uma auditoria de catálogo. Em `listProducts` o enum é `1` Últimos incluídos, `2` Ativos, `3` Inativos, `4` Excluídos, `5` Todos — **diferente** do enum de `listContacts`, não reaproveitar os números entre as duas tools. Atenção ao universo contado: `tipo=P` não traz variações (`V`) nem composições (`E`), então produtos pai devem ser complementados com `listProductVariations` quando a auditoria incluir variações. Se `listProducts` retornar timeout, prosseguir com os produtos recebidos e indicar "análise parcial: N produtos analisados". Mapear nas quatro categorias diagnósticas: (1) **Fiscal**: NCM, GTIN, origem ausentes; (2) **Descritiva**: descrição vazia ou curta, sem imagens; (3) **Dimensional**: peso ou dimensões zerados; (4) **Categorização**: sem categoria ou categoria incorreta. Cada categoria deve reportar contagem e percentual, **declarando qual universo foi contado** (ex.: "312 produtos `tipo=P`, variações não incluídas").

### 2. Estrutura de categorias e grupos
`listProductCategories` e `listProductGroups` → árvore atual. Propor reorganização quando houver produtos órfãos ou categorias redundantes.

### 3. Plano de mudança (obrigatório antes de qualquer escrita)
Apresentar tabela: produto → campo → valor atual → valor proposto → origem do valor proposto (informado pelo usuário, inferido de item semelhante, padrão da categoria). Valores fiscais (NCM/CEST) propostos por inferência devem ser marcados como **"validar com contador"** — nunca aplicados sem confirmação específica.

### 4. Execução em lote (após aprovação do plano)
- Correções: `patchProduct` por produto, somente os campos aprovados (patch parcial, nunca sobrescrever o cadastro inteiro)
- Estrutura: `createProductCategory`/`createProductGroup` conforme o plano
- Novos produtos: `createProduct` com checklist mínimo completo (nome, SKU, unidade, preço, NCM, GTIN quando houver) — recusar cadastrar às cegas um produto incompleto, que nasceria bloqueando o faturamento
- Registrar cada alteração aplicada; em erro num item, continuar o lote e reportar os que falharam no final

### 5. Relatório
Relatório nas quatro categorias diagnósticas (Fiscal, Descritiva, Dimensional, Categorização) com % de produtos afetados; antes/depois da completude (% com cadastro fiscal completo, % categorizados), alterações aplicadas, itens que falharam e pendências marcadas para o contador.

## Cuidados
- Nunca alterar preço em massa sem o usuário informar explicitamente a regra (percentual, tabela, lista).
- Duplicidades: apenas apontar — a decisão de inativar/fundir é do usuário.
- Integração natural: rodar depois do preflight-fiscal-natureza (que gera a fila de correção) e antes da esteira-faturamento-guiado.
