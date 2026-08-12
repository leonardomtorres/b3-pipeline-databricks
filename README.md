# B3 Ações Pipeline — Databricks & Spark

Sou apaixonado por mercado de ações, tenho curso oficial na B3 e já operei na bolsa. 
Quando comecei a estudar engenharia de dados, quis juntar as duas coisas num projeto real.

A ideia foi simples: pegar dados históricos de ações brasileiras, processar com Spark 
no Databricks e descobrir quem performou melhor desde 2022.

## O que o pipeline faz

Coleta dados de 10 ações da B3 via API → limpa e transforma → gera análises de 
retorno e volatilidade → plota os resultados → prepara features e treina um 
modelo pra tentar prever o retorno do dia seguinte.

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
volatilidade recente, retornos passados) e treinei um XGBoost pra tentar prever 
o retorno do dia seguinte, rastreando tudo com MLflow.

**Resultado:** MAE de ~1,48 e R² próximo de zero. Não teve milagre, e isso já era 
esperado — retorno diário de ação é praticamente ruído, então nenhum modelo simples 
bate isso de cara. O ponto aqui não foi "prever a bolsa", foi mostrar o fluxo 
completo: dado bruto → features → modelo treinado → experimento versionado.

## Notebooks

- `01_bronze_ingestion` — coleta via API e salva em Delta Lake
- `02_silver_transform` — limpeza e retorno diário com Window Functions
- `03_gold_analytics` — ranking de retorno e volatilidade por setor
- `04_visualization` — gráficos com matplotlib
- `05_feature_engineering` — médias móveis, volatilidade e features pro modelo
- `06_ml_training` — treino do XGBoost com tracking via MLflow

## Como rodar

1. Criar conta gratuita em databricks.com/learn/free-edition
2. Importar os notebooks e rodar na ordem 01 → 06 (célula por célula)
3. yfinance é instalado direto com `%pip install yfinance`, e o mesmo vale pro 
   `06_ml_training`, que instala `mlflow` e `xgboost` sozinho

## Próximos passos

- Hoje os notebooks rodam manualmente, um por um — orquestrar isso com 
  Databricks Jobs (ou Airflow) é o próximo passo natural pra deixar o pipeline 
  automatizado de ponta a ponta
