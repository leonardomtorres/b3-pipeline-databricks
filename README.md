# Pipeline de Ações da B3 com Databricks, Machine Learning e GenAI

Sou apaixonado por mercado de ações. Quando comecei a estudar Engenharia de Dados, quis juntar os dois temas em um projeto prático.

O projeto começou como um pipeline para coletar e analisar o histórico de dez ações brasileiras desde 2022. Depois, evoluiu para preparar features, treinar e comparar modelos de Machine Learning, acompanhar experimentos com MLflow e processar manchetes com um LLM no Databricks.

O objetivo não é criar uma estratégia automática de investimento. A proposta é mostrar a evolução de um dado desde a ingestão até a experimentação com ML e GenAI, mantendo claras as limitações de cada resultado.

## Arquitetura

Fluxo principal de dados estruturados:

```text
yfinance → Bronze → Silver → Gold → Feature Engineering → XGBoost → MLflow
```

Fluxo paralelo de dados não estruturados:

```text
Manchetes → Bronze → LLM → Sentimento → Silver → Agregação por ticker → Gold
```

- **Bronze:** recebe os dados brutos de preços e as manchetes usadas no experimento de GenAI.
- **Silver:** aplica tipagem e transformações, calcula retornos e estrutura o sentimento produzido pelo LLM.
- **Gold:** disponibiliza rankings, volatilidade por setor, features para ML e sentimento agregado por ticker.

Os notebooks são executados em sequência no Databricks. Cada etapa lê as tabelas Delta produzidas pelas anteriores.

## Fluxo do projeto

### 1. Engenharia de Dados

O `yfinance` coleta preços de abertura, máxima, mínima, fechamento e volume de dez ações. Os dados são convertidos para Spark e gravados em Delta Lake. Na camada Silver, Window Functions separam cada ticker em ordem cronológica para calcular retorno diário e retorno acumulado. A Gold consolida o ranking de retorno e a volatilidade por setor, que alimentam as visualizações do projeto.

### 2. Feature Engineering

No notebook `05_feature_engineering`, os dados tratados da Silver são transformados em uma tabela Gold preparada para modelagem, a `b3_pipeline.gold_ml_features`.

As features criadas são:

- médias móveis do preço de fechamento em 5 e 10 dias;
- volatilidade móvel do retorno em 5 dias;
- retornos defasados em 1, 3 e 5 dias (`lags`);
- retorno do dia seguinte como primeiro target;
- volatilidade do dia seguinte como segundo target.

Todas as janelas são particionadas por ticker e ordenadas por data. Isso evita misturar o histórico de ações diferentes e garante que as features usem somente observações coerentes com a série temporal.

### 3. Machine Learning e MLflow

No notebook `06_ml_training`, a tabela de features é convertida para pandas porque possui pouco mais de 10 mil linhas na execução atual, um volume pequeno para processamento em memória. Os 20% finais das datas formam o teste. Como o target de volatilidade olha cinco pregões à frente, uma faixa de segurança de cinco datas separa treino e teste. Não há embaralhamento aleatório, o que evita colocar informações futuras no treino e reduz o risco de *data leakage*.

Foram treinados dois modelos `XGBRegressor` com as mesmas features e targets diferentes:

| Experimento | Target | MAE | RMSE | R² |
|---|---|---:|---:|---:|
| XGBoost para retorno | `target_retorno_prox_dia` | 1,5288 | 2,0916 | -0,0117 |
| XGBoost para volatilidade | `target_volatilidade_futura_5d` | **0,6699** | **0,9015** | **0,1793** |
| Baseline de persistência | `target_volatilidade_futura_5d` | 0,8163 | 1,1178 | -0,2619 |

MAE, RMSE e R² avaliam os modelos sob perspectivas diferentes. O MAE mostra o erro absoluto médio na unidade do target. O RMSE penaliza mais os erros grandes. O R² indica quanto da variação do target foi explicada pelo modelo e pode ser negativo quando o desempenho é inferior ao uso da média como referência.

O modelo de retorno não superou uma referência baseada na média. Na volatilidade, o XGBoost reduziu o MAE em aproximadamente 18% e o RMSE em aproximadamente 19% em relação ao baseline de persistência. O R² de 0,1793 é moderado, mas o ganho sobre uma regra simples indica que as features carregaram algum sinal no período de teste.

Cada treinamento é registrado como um run separado no MLflow. Parâmetros, MAE, RMSE, R² e o artefato do modelo ficam associados ao experimento. As métricas do baseline também são registradas junto ao run de volatilidade.

### 4. GenAI e classificação de sentimento

O notebook `07_news_sentiment` adiciona dados não estruturados ao projeto. Um LLM disponível por endpoint no Databricks recebe um prompt e classifica cada manchete como `positivo`, `neutro` ou `negativo`.

Essa abordagem é uma **zero-shot sentiment classification**: o LLM não é treinado novamente com exemplos do projeto. Ele usa o conhecimento adquirido no pré-treinamento e segue a instrução fornecida no prompt. A resposta textual é normalizada e convertida em uma representação numérica:

```text
positivo = 1
neutro   = 0
negativo = -1
```

Depois, o resultado é salvo na Silver e agregado por ticker na tabela `b3_pipeline.gold_sentimento_ticker`. Assim, a saída do LLM deixa de ser apenas texto e passa a ter uma estrutura que pode ser integrada a análises ou modelos futuros.

As 24 manchetes desse notebook são fictícias e foram escritas apenas para fins educacionais. Elas estão identificadas como tal na coluna `nota`. O volume é suficiente para demonstrar o processamento com LLM, mas não para testar se sentimento melhora a previsão do mercado.

## Resultados da análise

![Retorno acumulado](docs/results/retorno_acumulado.png)

![Volatilidade por setor](docs/results/volatilidade_setor.png)

Na execução registrada no projeto:

- PETR4 apresentou o maior retorno acumulado desde 2022, acima de 300%;
- MGLU3 teve queda próxima de 90% no período;
- o varejo apareceu como o setor mais volátil;
- a previsão de retorno diário não superou uma referência simples baseada na média;
- o XGBoost de volatilidade reduziu o MAE em aproximadamente 18% frente ao baseline e obteve R² de 0,1793 no recorte de teste.

O principal aprendizado foi separar duas perguntas que parecem semelhantes, mas têm comportamentos diferentes. Prever a direção ou o retorno de uma ação não é o mesmo que estimar seu risco. O projeto também mostrou como transformar a saída textual de um LLM em dado estruturado dentro da mesma arquitetura em camadas.

## Notebooks

### Engenharia de Dados

| Notebook | Responsabilidade |
|---|---|
| `01_bronze_ingestion` | Coleta preços com `yfinance` e grava os dados brutos em Delta Lake. |
| `02_silver_transform` | Aplica tipagem e calcula retorno diário e acumulado com Window Functions. |
| `03_gold_analytics` | Gera o ranking de retorno e a volatilidade por setor. |
| `04_visualization` | Cria os gráficos com matplotlib. |

### Machine Learning

| Notebook | Responsabilidade |
|---|---|
| `05_feature_engineering` | Cria médias móveis, volatilidade, lags e targets futuros sem sobreposição entre as janelas de volatilidade. |
| `06_ml_training` | Treina os dois modelos XGBoost, aplica uma separação temporal com faixa de segurança, compara com baseline e registra os runs no MLflow. |

### GenAI

| Notebook | Responsabilidade |
|---|---|
| `07_news_sentiment` | Usa um LLM para classificar manchetes e grava o sentimento estruturado em Silver e Gold. |

## Tecnologias utilizadas

- Python, pandas e matplotlib
- PySpark e Spark SQL
- Delta Lake e arquitetura Bronze, Silver e Gold
- Databricks Free Edition
- yfinance
- XGBoost e scikit-learn
- MLflow
- Databricks Model Serving / Foundation Model API
- LLM, prompt engineering e zero-shot classification

## Como executar

1. Crie um workspace no [Databricks Free Edition](https://www.databricks.com/learn/free-edition).
2. Importe os notebooks da pasta `notebooks`.
3. Execute os notebooks em ordem, do `01` ao `07`.
4. No notebook `07`, execute primeiro a listagem de endpoints e confirme o nome do LLM disponível no workspace antes de iniciar a classificação.

As dependências são instaladas nos próprios notebooks com `%pip install`. As tabelas são sobrescritas durante a execução, portanto uma versão atualizada de um notebook deve ser executada antes dos notebooks que dependem dela.

## Limitações e próximos passos

- A execução ainda é manual. Databricks Jobs ou Airflow poderiam organizar dependências, tentativas e agendamento.
- A avaliação dos modelos usa um único corte temporal de 80/20. Uma validação *walk-forward* daria uma visão mais consistente do desempenho ao longo de diferentes períodos.
- Os resultados refletem dez ações e o período iniciado em 2022. Eles não devem ser interpretados como recomendação de investimento.
- O XGBoost de volatilidade superou o baseline no corte atual, mas ainda precisa ser validado em outros períodos antes de qualquer conclusão sobre generalização.
- As manchetes são fictícias e pouco numerosas. Um experimento estatístico exigiria notícias reais, mais observações e alinhamento temporal com os pregões.
- A tabela Gold de sentimento está pronta para integração, mas o notebook atual ainda não a adiciona às features nem retreina os modelos com essa informação.
