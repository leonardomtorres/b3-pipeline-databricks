# B3 Ações Pipeline — Databricks & Spark

Sou apaixonado por mercado de ações, tenho curso oficial na B3 e já operei na bolsa. 
Quando comecei a estudar engenharia de dados, quis juntar as duas coisas num projeto real.

A ideia foi simples: pegar dados históricos de ações brasileiras, processar com Spark 
no Databricks e descobrir quem performou melhor desde 2022.

## O que o pipeline faz

Coleta dados de 10 ações da B3 via API → limpa e transforma → gera análises de 
retorno e volatilidade → plota os resultados.

## Stack

- Databricks Community Edition
- PySpark + Delta Lake
- yfinance API
- Python e Spark SQL

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

## Notebooks

- `01_bronze_ingestion` — coleta via API e salva em Delta Lake
- `02_silver_transform` — limpeza e retorno diário com Window Functions
- `03_gold_analytics` — ranking de retorno e volatilidade por setor
- `04_visualization` — gráficos com matplotlib

## Como rodar

1. Criar conta gratuita em community.cloud.databricks.com
2. Rodar os notebooks na ordem 01 → 04
3. yfinance é instalado direto com `%pip install yfinance`
