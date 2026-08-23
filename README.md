# B3 Ações Pipeline — Databricks & Spark

Sou apaixonado por mercado de ações. 
Quando comecei a estudar engenharia de dados, quis juntar as duas coisas num projeto real.

A ideia foi simples: pegar dados históricos de ações brasileiras, processar com Spark 
no Databricks e descobrir quem performou melhor desde 2022.

## O que o pipeline faz

Coleta dados de 10 ações da B3 via API → limpa e transforma → gera análises de 
retorno e volatilidade → plota os resultados → prepara features e testa duas 
hipóteses de Machine Learning: dá pra prever o retorno de amanhã? E a volatilidade?

## Stack

- Databricks Free Edition
- PySpark + Delta Lake
- yfinance API
- Python e Spark SQL
- pandas, scikit-learn, XGBoost
- MLflow (tracking de experimentos)

## Arquitetura

Bronze → Silver → Gold

- **Bronze** — dados brutos da API, sem transformação
- **Silver** — limpeza, tipagem e retorno diário calculado
- **Gold** — ranking de retorno acumulado e volatilidade por setor

## Resultados

![Retorno acumulado](docs/results/retorno_acumulado.png)

![Volatilidade por setor](docs/results/volatilidade_setor.png)

**O que encontrei:**
- PETR4 liderou com +310% de retorno desde 2022
- MGLU3 perdeu 90% do valor no mesmo período
- Varejo foi o setor mais volátil — 2,5x mais que telecom
- Quem opera na bolsa já sente isso, mas é diferente provar com dados

## Modelagem

Depois de fechar a parte de engenharia, fui um passo além e usei os dados prontos 
pra montar uma camada de ciência de dados: criei features (médias móveis, 
volatilidade recente, retornos passados) e treinei dois modelos XGBoost, cada um 
testando uma hipótese diferente, rastreados com MLflow.

| Hipótese | Alvo | MAE | R² |
|---|---|---|---|
| Dá pra prever o **retorno** de amanhã? | `target_retorno_prox_dia` | 1,48 | -0,01 |
| Dá pra prever a **volatilidade** de amanhã? | `target_volatilidade_prox_dia` | 0,31 | **0,76** |

**O que isso significa:** retorno diário de ação é praticamente ruído — nenhum 
modelo simples bate isso de cara, e um R² perto de zero aqui é o resultado 
esperado, não uma falha. Volatilidade é outra história: existe um fenômeno bem 
documentado em finanças chamado *volatility clustering* (dia volátil tende a ser 
seguido de outro dia volátil), e o modelo capturou isso bem — um R² de 0,76 é um 
resultado forte pra esse tipo de problema.

O valor do projeto não foi "prever a bolsa" — foi testar duas hipóteses com dados 
reais e deixar os números falarem: uma confirmou que preço futuro é difícil de 
prever, a outra confirmou que risco futuro (volatilidade) não é.

## Notebooks

- `01_bronze_ingestion` — coleta via API e salva em Delta Lake
- `02_silver_transform` — limpeza e retorno diário com Window Functions
- `03_gold_analytics` — ranking de retorno e volatilidade por setor
- `04_visualization` — gráficos com matplotlib
- `05_feature_engineering` — médias móveis, volatilidade e os dois targets (retorno e volatilidade)
- `06_ml_training` — treino dos dois modelos XGBoost, com tracking via MLflow

## Como rodar

1. Criar conta gratuita em databricks.com/learn/free-edition
2. Importar os notebooks e rodar na ordem 01 → 06 (célula por célula)
3. yfinance é instalado direto com `%pip install yfinance`, e o mesmo vale pro 
   `06_ml_training`, que instala `mlflow` e `xgboost` sozinho

## Próximos passos

- Hoje os notebooks rodam manualmente, um por um — orquestrar isso com 
  Databricks Jobs (ou Airflow) é o próximo passo natural pra deixar o pipeline 
  automatizado de ponta a ponta
