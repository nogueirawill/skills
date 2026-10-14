---
name: otimizador-anuncios-sku
description: "Otimizador de Anúncios e Cadastro de SKU via MCP Bling. Audita título, descrição, GTIN/EAN, NCM, imagens, dimensões e preço de um produto, cruza com o histórico de vendas e sugere título SEO para marketplace e correções de cadastro. Use sempre que o usuário mencionar otimizar anúncio, melhorar título, SEO de marketplace, cadastro incompleto, buybox, precificação de SKU, elasticidade de preço, ou pedir \"Otimização de SKU\"."
---

# Otimizador de Anúncios e Preços (SKU)

Melhora conversão atacando as duas frentes: qualidade do cadastro e precificação baseada em histórico.

## Pré-requisitos

Tools deferidas — execute `tool_search` com `"MCP Bling listProducts getProduct listProductVariations getStockBalance listSalesOrders getSalesOrder listProductCategories patchProduct"` (limit: 20). Buscar de novo pelo nome exato se faltar alguma. **Tools usadas:** `listProducts`, `getProduct`, `listProductVariations`, `getStockBalance`, `listSalesOrders` + `getSalesOrder` (histórico do SKU), `listProductCategories`, `patchProduct` (escrita — só com confirmação).

Ao chamar qualquer tool do MCP, preencher o campo `explanation` com `[skill:otimizador-anuncios-sku] <raciocínio da chamada>` — o prefixo identifica a skill de origem na observabilidade e o texto seguinte preserva o raciocínio que o campo pede. Se a tool não expuser `explanation`, não enviar o campo.

**Paginação (importante para qualquer número agregado):** por padrão as listagens retornam **apenas uma página** (até 100 registros), e o `_meta` da resposta traz `has_more`, `pagina_atual` e `proxima_pagina`. Consequências:

- Para **contagens, somas, rankings, share, médias e taxas**, passar `buscar_tudo: true` na chamada. Sem isso o total é calculado sobre 100 registros e sai errado com aparência de certo.
- Mesmo com `buscar_tudo: true` a varredura pode parar antes do fim (teto de páginas ou tempo limite do servidor). **Sempre conferir `_meta.buscaCompleta`**: quando vier `false`, declarar no relatório que a cobertura é parcial, informando `paginasBuscadas` e `totalRegistrosBruto`, e não apresentar o número como total do período.
- Para **amostra ou navegação**, manter o padrão de uma página e, se precisar de mais, chamar de novo incrementando `pagina` enquanto `has_more` for `true`. Filtrar (por data, situação, loja, código) é sempre preferível a varrer tudo.

Antes de filtrar ou apresentar dados por situação, ler o resource `bling://guia/situacoes`: ele traz os enums fixos (NF-e/NFC-e, contas a pagar e a receber, pedidos de compra, propostas), o procedimento de discovery para as entidades com IDs dinâmicos por conta (pedidos de venda, ordens de produção) e as regras de apresentação por tipo de consulta (histórico, ranking, operacional, diagnóstico). Não reconstruir esse mapa por conta própria — usar `listStatusModules` apenas para o que o guia indicar como dinâmico.

## Fluxo de execução

### 1. Localizar o produto
`listProducts(nome | codigos: [SKU], fields: [...])` → obter o `id`. Se o ID fornecido pelo usuário retornar HTTP 404 em `getProduct`, interromper a análise, reportar "produto não encontrado" e listar os 5 primeiros resultados de `listProducts` como alternativas. Se for produto pai com variações, `listProductVariations(idProduto)` para auditar cada variação.

### 2. Auditoria de cadastro (`getProduct`)
Checklist de completude — marcar ✅/❌ para:
- **Título** (`nome`): usa os 120 caracteres? Segue padrão marketplace `Produto + Marca + Modelo + Atributo-chave + Tamanho/Cor`? Sem termos internos tipo "NÃO OFERTAR"/"teste"?
- **GTIN/EAN** (`gtin`) — obrigatório na maioria dos marketplaces
- **NCM/CEST** (`tributacao.ncm`, `cest`) — trava emissão de NF se ausente
- **Descrição** (`descricaoCurta`/`descricaoComplementar`) — vazia ou pobre?
- **Imagens** (`midia`/`imagemURL`), **dimensões e pesos** (frete correto), **marca**, **categoria** (`listProductCategories`), **preço** (`preco`) e **custo**. Atenção ao caminho do custo: em `getProduct` ele está em **`fornecedor.precoCusto`**, não na raiz — na raiz o campo só existe na listagem (`listProducts` com `fields: ["precoCusto"]`). Ler da raiz do `getProduct` sempre parece vazio, e aí a análise de preço conclui "sem custo" para produto que tem custo cadastrado. Sem custo real, não estimar margem: marcar como pendência de cadastro.

  `precoCusto` é o custo do **fornecedor padrão cadastrado**, não o custo médio ponderado do estoque (que soma entradas reais com frete e impostos rateados). A API v3 não expõe o custo médio, então a análise de preço desta skill usa a melhor aproximação disponível — não o número contábil. Se o usuário questionar divergência entre a margem sugerida e o que a contabilidade calcula, dizer isso em vez de defender o valor.

### 3. Performance de venda
`listSalesOrders` dos últimos 90 dias + `getSalesOrder` dos pedidos relevantes, filtrando itens do SKU: unidades, receita, preço médio praticado, descontos aplicados. Se houve variação de preço no período, comparar volume antes/depois (elasticidade observada — deixar claro que é correlação com amostra pequena, não prova causal).

### 4. Relatório (formato de saída)
1. **Scorecard do cadastro** (nota 0–10 + checklist ✅/❌)
2. **Título otimizado sugerido** (até 60 caracteres para Meli + versão estendida) e descrição com bullets de benefício. Se dados insuficientes (produto não encontrado ou campos vazios), gerar título-template: "[Categoria] [Tipo de Produto] — [Característica Principal]" com placeholders explícitos.
3. **Análise de preço**: preço atual vs. preço médio vendido vs. custo (margem bruta) e recomendação
4. **Pendências bloqueantes** (GTIN/NCM ausentes etc.)
5. Estoque atual via `getStockBalance` (anúncio otimizado sem estoque = problema maior)

### 5. Ação opcional
Aplicar correções com `patchProduct(idProduto, body)` — **somente os campos aprovados pelo usuário, campo a campo**, mostrando antes/depois antes de executar. Confirmação genérica ("pode corrigir tudo") não é suficiente — listar cada campo proposto e aguardar confirmação individual. Nunca alterar preço sem confirmação explícita do valor.

## Cuidados
- Para listas grandes ("audite todos os produtos"), rodar em modo amostral (top 20 por venda) e oferecer aprofundamento.
- Sugestões de SEO são para o cadastro no Bling; a sincronização com cada marketplace segue as regras da integração configurada.
- **Buy Box, posição no ranking do marketplace, preço de concorrente e reputação do anúncio não existem na API do Bling.** Quando o usuário pedir "por que perdi o buybox", explicar que o dado não é consultável por aqui e entregar o que dá para atacar de fato — completude do cadastro, título, preço vs. custo e disponibilidade de estoque, que são fatores de Buy Box. Nunca estimar posição, share de Buy Box ou preço de concorrente.
