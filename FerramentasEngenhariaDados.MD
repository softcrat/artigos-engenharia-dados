# 🗺️ Mapeamento de Arquitetura de Dados: Nuvem vs. Open Source

Este guia apresenta um comparativo rápido das principais ferramentas e serviços utilizados em uma **Pipeline de Dados** moderna, segmentado por camadas arquiteturais. É uma referência útil para identificar as equivalências entre as soluções de **AWS, GCP, Azure** e suas respectivas **alternativas Open Source** para fins de aprendizado e prototipagem.

---

## 🌐 Camada de Ingestão de Dados

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source (para aprender) | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Mensageria / Streaming** | Amazon Kinesis | Pub/Sub | Event Hubs | 🟩 **Apache Kafka** / Redpanda | Padrão de mercado; a maioria dos serviços *cloud* de *streaming* se baseia no Kafka. |
| **Filas Simples** | SQS | Pub/Sub Lite | Azure Queue Storage | 🟩 RabbitMQ, ActiveMQ, NATS | Ideal para entender filas e padrões de mensageria. |
| **Ingestão de Arquivos** | S3 Upload, Glue Crawler | Cloud Storage, Dataflow | Azure Blob Storage | 🟩 MinIO, Apache NiFi, Airbyte, Singer | **MinIO** simula S3; **NiFi** e **Airbyte** para ingestão automatizada. |
| **ETL/ELT Low-Code** | AWS Glue DataBrew | DataPrep | Data Factory | 🟩 Pentaho (PDI), Apache Hop, Meltano | Boas para praticar ETL visual. |

---

## ⚙️ Camada de Processamento / Transformação

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Processamento Distribuído (Batch)** | EMR (Spark) | Dataproc (Spark) | HDInsight (Spark) | 🟩 **Apache Spark** (standalone ou via Docker) | É o motor padrão da indústria para *batch*. |
| **Processamento em Fluxo (Streaming)** | Kinesis Data Analytics | Dataflow (Beam) | Stream Analytics | 🟩 **Apache Flink**, Apache Beam, Spark Structured Streaming | Aprenda **Flink** ou **Beam**, pois são compatíveis com os serviços *cloud*. |
| **SQL Interativo** | Athena (Presto) | BigQuery | Synapse Serverless SQL | 🟩 **Trino** (PrestoSQL), DuckDB | **Trino** é quase idêntico ao Athena; **DuckDB** é ótimo localmente. |
| **Orquestração de Pipelines** | Step Functions, MWAA (Airflow) | Composer (Airflow) | Data Factory | 🟩 **Apache Airflow**, Prefect, Dagster | **Airflow** é usado amplamente em nuvem (AWS, GCP). |

---

## 💾 Camada de Armazenamento e Data Lake

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Object Storage** | S3 | Cloud Storage | Blob Storage | 🟩 **MinIO**, Ceph, SeaweedFS | **MinIO** usa API 100% S3-compatible. |
| **Data Warehouse** | Redshift | BigQuery | Synapse Analytics | 🟩 **ClickHouse**, DuckDB, Druid, PostgreSQL + Citus | Simulam bem conceitos de DW (paralelismo, colunaridade). |
| **Metastore / Catálogo** | AWS Glue Catalog | Dataplex / Data Catalog | Purview | 🟩 **Hive Metastore**, Amundsen, DataHub, OpenMetadata | Metadados e *lineage* compatíveis com Spark, Delta, Hive. |
| **Formato de Tabela** | Delta Lake (Databricks AWS) | BigLake | Synapse Tables | 🟩 **Delta Lake**, Apache Iceberg, Apache Hudi | Ferramentas *open-source* idênticas às usadas em nuvem. |

---

## 📊 Camada de Consumo / BI

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **BI / Dashboard** | QuickSight | Looker / Data Studio | Power BI | 🟩 **Metabase**, Apache Superset, Redash, Grafana | **Metabase** e **Superset** são gratuitos e simples de usar. |
| **SQL Engine para Análise** | Athena / Redshift | BigQuery | Synapse | 🟩 **Trino**, DuckDB, Presto | Úteis para consultas *ad-hoc*. |

---

## 🔒 Camada de Segurança / Governança

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Gestão de Permissões** | IAM | Cloud IAM | Azure AD | 🟩 Keycloak, Open Policy Agent (OPA) | Conceitos idênticos (**RBAC, SSO, tokens, policies**). |
| **Auditoria / Catalogação** | AWS Glue Catalog + CloudTrail | Dataplex / Cloud Audit Logs | Purview | 🟩 DataHub, Amundsen, OpenMetadata | São catálogos *open-source* focados em *lineage* e auditoria. |

---

## 🔁 Camada de Orquestração e Observabilidade

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Orquestração** | Step Functions, MWAA (Airflow) | Composer | Data Factory | 🟩 **Airflow**, Prefect, Dagster | **Airflow** é compatível com AWS e GCP. |
| **Monitoramento de Pipelines** | CloudWatch, Stackdriver | Monitoring / Logging | Application Insights | 🟩 **Prometheus**, Grafana, OpenTelemetry | Monitoração de *jobs* e métricas. |

---

## 🔧 Camada de Deploy / Infraestrutura

| Conceito | Cloud AWS | Cloud GCP | Cloud Azure | Alternativas Open Source | Observação |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Infraestrutura como Código** | CloudFormation, CDK | Deployment Manager | ARM Templates / Bicep | 🟩 **Terraform**, Pulumi, Ansible | **Terraform** é padrão *multi-cloud*. |
| **Contêiner / Orquestração** | ECS / EKS | GKE | AKS | 🟩 **Docker Compose**, Kubernetes (Kind / Minikube) | Permite simular *deploys* locais idênticos aos da nuvem. |
