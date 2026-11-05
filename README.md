# Antecipação de Recebíveis da Olist

Estudo de antecipação de recebíveis sobre o dataset público da Olist (Kaggle): pipeline bronze→silver→gold em PySpark e SQL no Databricks, análise da agenda em aberto (R$ 2,4 mi), concentração, cohort e segmentação de lojistas - regra de negócio validada com k-means, evoluindo de 4 para 5 segmentos.

## Sobre o projeto

Marketplaces que vendem no crédito parcelado geram recebíveis futuros para o lojista: a parcela _k_ de uma venda liquida em **D + 30·k**. Enquanto isso, o lojista espera. Antecipar essa agenda é o produto estudado aqui — e a **agenda em aberto é o estoque desse produto**.

A análise responde a três perguntas:

1. **Onde está a agenda?** (concentração, categorias, prazo de espera)
2. **Os lojistas se comportam como a segmentação por regra assume?** (validação com k-means)
3. **Como tratar cada grupo?** (hipóteses de produto por segmento)

## Stack

- **Databricks** — notebooks, Unity Catalog (catálogo `workspace.olist`)
- **PySpark e SQL** — pipeline bronze → silver → gold
- **pandas, scikit-learn, matplotlib** — EDA, k-means, PCA e visualizações

## Dataset

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle): pedidos, pagamentos, clientes, lojistas e reviews de um marketplace brasileiro.

- Data de corte: **2018-06-30** · janela de análise: **6 meses**
- Regra da agenda: a parcela _k_ de uma venda no crédito liquida em D+30·k; "em aberto" é tudo com liquidação após a data de corte

## Estrutura do projeto

| Notebook                                       | Papel                                                                                     |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `antecipacao_recebeis_bronze-silver`           | Ingestão do dataset bruto e limpeza/validação na silver                                   |
| `antecipacao_recebeis_gold`                    | Construção das tabelas gold (fato de agenda de recebíveis)                                |
| `antecipacao_recebeis_gold_diagnostico`        | EDA: Pareto, mix de pagamento, categorias, cohort e segmentação por regra (5 segmentos)   |
| `antecipacao_recebeis_gold_kmeans_segmentacao` | Clustering k-means (k por silhouette), PCA, ARI vs. regra e nomes de negócio dos clusters |
| `antecipacao_recebeis_gold_deck`               | Deck de apresentação: 6 slides de negócio + apêndice técnico, recalculado a cada execução |

### Tabelas gold

| Tabela                  | Conteúdo                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| `gold_seller_metricas`  | Uma linha por lojista: TPV, mix de pagamento, agenda em aberto, prazo, ticket, meses ativos |
| `gold_seller_segmento`  | Segmento por regra de cada lojista (5 segmentos)                                            |
| `gold_segmento_resumo`  | Perfil agregado por segmento                                                                |
| `gold_categoria_resumo` | Mix, agenda e prazo por categoria de produto                                                |
| `gold_cohort_sellers`   | Atividade por cohort (mês da primeira venda)                                                |
| `gold_seller_cluster`   | Cluster k-means de cada lojista qualificado + variáveis usadas                              |
| `gold_cluster_perfil`   | Perfil agregado por cluster, com k e ARI vs. regra                                          |

## Segmentação por regra (avaliada em ordem)

| Ordem | Segmento                | Critério                                                                             |
| ----- | ----------------------- | ------------------------------------------------------------------------------------ |
| 1     | Grandes parceladores    | Top 20% por agenda em aberto **e** > 50% do TPV parcelado                            |
| 2     | À vista                 | < 20% do TPV parcelado                                                               |
| 3     | Médios recorrentes      | Vendas em pelo menos 3 dos últimos 6 meses                                           |
| 4     | Grandes ocasionais      | Agenda em aberto **e** > 50% do TPV parcelado **e** prazo médio contratado ≥ 90 dias |
| 5     | Pequenos ou esporádicos | O restante                                                                           |

O 4º critério (**Grandes ocasionais**) foi aberto depois da validação por k-means: o cluster "Grandes vendas, longa espera" aparecia espalhado entre "Pequenos ou esporádicos" e "Grandes parceladores" — esporádicos no calendário, mas grandes na venda.

## Validação com k-means

- Base: **1.875 de 2.717 lojistas** (69%) — exigência de ≥ 3 pedidos para reduzir ruído
- Variáveis padronizadas: log(agenda), prazo médio contratado, log(ticket), meses ativos, % TPV parcelado
- **k = 3** pela maior silhouette (0,291)
- Clusters: `Recorrentes com agenda` (864 lojistas, 61% da agenda) · `Grandes vendas, longa espera` (473, 38%) · `Baixa atividade` (538, 0,8%)
- ARI regra vs. k-means: **0,27 → 0,30** após a abertura do 5º segmento

## Principais achados

- **Agenda em aberto: R$ 2,43 mi** para 1.751 dos 2.717 lojistas (64%), com prazo ponderado de **90 dias**
- **Concentração extrema**: "Grandes parceladores" são 17% dos lojistas e carregam 78% da agenda → dois trilhos: negociação ativa para os grandes, self-service para a cauda
- **Eletrônicos e móveis dominam a espera**: categorias com 95–147 dias de prazo contratado e 60–83% do TPV parcelado
- **Recorrência é exceção**: 58% dos lojistas ativos no mês 1, 46% no mês 6
- **5º segmento (Grandes ocasionais)**: 234 lojistas, R$ 59,9 mil de agenda, ~115 dias de espera, ~1 mês ativo — antecipação com limite inicial conservador

## Como reproduzir

1. Importar o dataset do Kaggle para o storage do workspace
2. Executar os notebooks na ordem da tabela acima (os nomes de tabelas e catálogo estão hardcoded como `workspace.olist`)
3. Todos os cortes (data, segmentos, k-means) ficam na célula **Regras de negócio** de cada notebook — ajuste ali e reexecute
4. O deck recalcula automaticamente os gráficos a partir das tabelas gold

## Limitações

- Não assume taxa, MDR ou custo de captação — "antecipar" é avaliado por valor e espera, não por margem
- Dados até 2018-06-30, janela de 6 meses: sazonalidade e crescimento posteriores não capturados
- "Grandes ocasionais" tem histórico curto (~1 mês ativo): qualquer limite de crédito precisaria começar conservador
