# Pipeline de Dados do Telegram com AWS

## Visão Geral do Projeto

Em um mundo digital dominado pela comunicação instantânea, os chatbots em plataformas como o Telegram tornaram-se ferramentas poderosas para interação automatizada e coleta de dados. Este projeto implementa um pipeline robusto e escalável na AWS (Amazon Web Services) para transformar a comunicação transacional de um grupo do Telegram em dados analíticos valiosos.

A motivação principal reside na diferença entre **Dados Transacionais** e **Dados Analíticos**:

| Tipo de Dado | Característica Principal | Exemplo no Projeto |
| :--- | :--- | :--- |
| **Transacional (Raw)** | Operacional, alta granularidade, foco na escrita rápida e integridade imediata. | Cada mensagem enviada no Telegram (Payload JSON). |
| **Analítico (Enriched)** | Histórico, sumarizado, estruturado (Parquet), foco na leitura otimizada e eficiência de consulta. | Volume de mensagens por hora, média de caracteres por usuário. |

O pipeline atua como uma ponte, movendo e transformando os dados brutos do Telegram para um formato analítico otimizado (Parquet particionado) na AWS, viabilizando análises profundas de comportamento e engajamento do grupo.

## Arquitetura do Pipeline

A arquitetura do projeto é dividida em dois grandes sistemas: o **Sistema Transacional** (Telegram), responsável pela geração de dados, e o **Sistema Analítico** (AWS), responsável pelo processamento, armazenamento e consulta.

*(Diagrama de Arquitetura Conceitual)*

### 1. 💬 Sistema Transacional (Telegram)

O Telegram, através da sua Bot API e do mecanismo de Webhook, atua como a fonte de dados. Cada mensagem enviada em um grupo monitorado é um evento que dispara o processo de ingestão.

#### Configuração do Bot

| Etapa | Descrição |
| :--- | :--- |
| **Criação do Bot** | O bot `will_ebac_bot` foi criado e configurado via `@BotFather`, obtendo o token necessário. |
| **Definição de ADM** | O bot foi promovido a Administrador do grupo para garantir que possa receber e processar todas as mensagens. |
| **Configuração de Privacidade** | A opção para adicionar o bot a novos grupos foi desabilitada, focando sua operação apenas no grupo de destino. |

### 2. ☁️ Sistema Analítico (AWS)

O sistema analítico é a parte mais robusta do pipeline, composto pelas etapas de Ingestão (Raw), ETL (Enriched) e Apresentação (Análise).

#### 2.1. Ingestão (Raw Data - Near Real-Time)

Esta etapa recebe o payload JSON do Telegram e o armazena imediatamente no formato original, no S3. A comunicação é feita via Webhook integrado ao API Gateway.

| Componente | Configuração / Ação |
| :--- | :--- |
| **Bucket S3** | Criação do bucket `project-pipeline-ebac-raw` para armazenamento dos dados brutos (JSON). |
| **Função Lambda (Raw)** | Função `project-pipeline-ebac-raw` em Python. Utiliza `boto3` para escrever o JSON do evento diretamente no S3, particionando por data (`context_date=YYYY-MM-DD`). |
| **API Gateway** | Criação de um método `POST` na API Gateway, servindo como endpoint HTTP para o Webhook do Telegram e integrado à Função Lambda Raw. |
| **IAM** | Anexação da política `AmazonS3FullAccess` ao role IAM para permitir a escrita no S3. |

#### 2.2. ETL (Enriched Data - Batch Diário)

Esta etapa de Transformação (T) e Carregamento (L) é agendada para rodar uma vez ao dia. Ela lê os múltiplos arquivos JSON brutos do dia anterior, aplica transformações de limpeza/estruturação, e salva o resultado no formato otimizado para análise.

| Componente | Configuração / Ação |
| :--- | :--- |
| **Função Lambda (Enriched)**| O código Python implementa a função `parse_data` (achatar e tipar o JSON) e `lambda_handler` (consolidar JSONs em um único Parquet). |
| **Layer PyArrow** | Criação de uma Layer Lambda com a biblioteca `pyarrow` para habilitar a manipulação e escrita do formato Parquet dentro do ambiente Lambda. |
| **Formato de Saída** | Os dados são consolidados em um único arquivo Parquet, particionado por data (`context_date`), e armazenado no bucket `project-pipeline-ebac-enriched`. |
| **EventBridge (Scheduler)**| Criação de uma regra de agendamento (cron) para invocar a função Lambda ETL diariamente, garantindo a execução do processo de compactação. |

#### 2.3. Apresentação (Análise Exploratória de Dados - EDA)

A etapa final utiliza o Amazon Athena para consultar os dados Parquet particionados diretamente no S3, empregando SQL para extrair insights de negócio.

| Etapa | Ação |
| :--- | :--- |
| **Criação da Tabela Athena**| Definição da Tabela Externa `telegram` no Athena, mapeando o schema do Parquet e configurando o particionamento por `context_date`. |
| **Adicionando Partições** | Execução do comando `MSCK REPAIR TABLE telegram;` para que o Athena reconheça as partições de dados criadas pelo Lambda no S3. |

## 📊 Análises de Exemplo (Insights)

A otimização dos dados para o Athena permite a execução de consultas complexas de forma eficiente:

#### Volume de Mensagens Diário:

```sql
SELECT 
    context_date,
    COUNT(message_id) AS message_amount
FROM telegram
GROUP BY 1
ORDER BY 1 DESC;
```

#### Média do Tamanho das Mensagens por Usuário:

```sql
SELECT
    context_date,
    user_first_name,
    CAST(AVG(CHAR_LENGTH(message_text)) AS INT) AS avg_message_length
FROM telegram
WHERE context_date = '2025-10-29'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

#### Frequência de Mensagens por Hora/Semana (Engajamento):

```sql
SELECT
    CAST(EXTRACT(HOUR FROM message_date) AS VARCHAR) AS message_hour,
    DAY_OF_WEEK(message_date) AS day_of_week,
    WEEK(message_date) AS week_number,
    COUNT(message_id) AS message_count
FROM telegram
GROUP BY 1, 2, 3
ORDER BY 3, 2, 1;
```

# 🛠️ Tecnologias Utilizadas

| Categoria| Tecnologia| Uso Principal
| :--- | :--- | :--- |
| Fonte de Dados| Telegram Bot API| Ingestão de mensagens via Webhook.
| Orquestração/Processamento| AWS Lambda (Python)| Funções Raw (Ingestão) e Enriched (ETL).
| Gateway/Endpoint| AWS API Gateway| Exposição do endpoint HTTP para o Webhook do Telegram.
| Armazenamento| AWS S3| Data Lake (armazenamento de dados Raw e Enriched).
| Agendamento| Amazon EventBridge| Configuração do scheduler (cron) para execução diária do ETL.
| Análise/Consulta| Amazon Athena| Query engine para análise exploratória dos dados Parquet.
| Bibliotecas| "boto3 pyarrow pandas"| Interação com serviços AWS e manipulação/escrita do formato Parquet.
