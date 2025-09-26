
# Pipeline Spark: ler do SAP HANA e gravar no PostgreSQL (otimizado para 20M de registros)

Objetivo: extrair 20M de registros do SAP HANA, transformar (se necessário) e gravar no PostgreSQL com throughput e confiabilidade.

### Estratégia geral

* Ler paralelamente do HANA usando JDBC com 'partitionColumn' + 'lowerBound' + 'upperBound' + 'numPartitions'.
* Aplicar transformações em Spark (cache/persist quando reusado).
* Reparticionar o DataFrame antes da escrita para controlar paralelismo.
* Escrever em PostgreSQL com 'batchsize' e 'isolationLevel' para reduzir overhead; para upsert usar staging table + 'MERGE'/'INSERT ON CONFLICT'.
* Monitorar e ajustar 'spark.sql.shuffle.partitions' e recursos do cluster.

### Dependências

* Driver JDBC do HANA (jar) disponível aos executores.
* Driver JDBC do PostgreSQL disponível aos executores.
* Spark configurado com memória/CPU suficiente.

### Exemplo PySpark (arquivo 'job_hana_to_pg.py')

'''python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col
import math

# Ajuste conforme seu cluster
spark = SparkSession.builder \
    .appName("hana_to_postgres_20M") \
    .config("spark.sql.shuffle.partitions", "200") \
    .getOrCreate()

# Credenciais / endpoints
hana_jdbc = "jdbc:sap://hana-host:30015"
hana_props = {
    "user": "HANA_USER",
    "password": "HANA_PASS",
    "driver": "com.sap.db.jdbc.Driver",
    # fetch size can help streaming reads
    "fetchsize": "10000"
}

pg_jdbc = "jdbc:postgresql://pg-host:5432/dbname"
pg_props = {
    "user": "PG_USER",
    "password": "PG_PASS",
    "driver": "org.postgresql.Driver",
    # important: tune batchsize
    "batchsize": "10000",
    "isolationLevel": "NONE"  # avaliar conforme sua política
}

# --- 1) Estimar bounds para particionamento (usar coluna numérica/PK)
# Supondo que a tabela tenha uma coluna id numérica e contínua
partition_col = "id"
table_hana = "SCHEMA.TABLE_HANA"

# opcional: ler min/max para particionar (uma leitura rápida)
bounds_q = f"(select min({partition_col}) as min_id, max({partition_col}) as max_id from {table_hana}) as bounds"
bounds_df = spark.read.jdbc(url=hana_jdbc, table=bounds_q, properties=hana_props)
bounds = bounds_df.collect()[0]
min_id, max_id = bounds["min_id"], bounds["max_id"]
total = None
if min_id is None or max_id is None:
    raise RuntimeError("Não foi possível determinar bounds para particionamento.")

# número de partições: balanceie entre paralelismo e overhead.
# Para 20M linhas, algo entre 100-400 partições costuma funcionar dependendo do cluster.
num_partitions = 200

# --- 2) Ler paralelamente do HANA
jdbc_table = table_hana  # ou consulta SQL entre parênteses como subquery alias
df = spark.read.jdbc(
    url=hana_jdbc,
    table=jdbc_table,
    column=partition_col,
    lowerBound=str(min_id),
    upperBound=str(max_id),
    numPartitions=num_partitions,
    properties=hana_props
)

# opcional: filtros iniciais / projeção para reduzir dados
df = df.select("id", "col_a", "col_b", "data_evento") \
       .filter(col("data_evento").isNotNull())

# Persistir caso vá reusar
df = df.repartition(200, col("id"))  # repartition antes do write para paralelismo desejado

# --- 3) Escrita: estratégia segura e performática
# Opção A: escrita paralela direta (append) com batchsize
df.write \
  .format("jdbc") \
  .option("url", pg_jdbc) \
  .option("dbtable", "schema.target_table") \
  .option("user", pg_props["user"]) \
  .option("password", pg_props["password"]) \
  .option("batchsize", pg_props["batchsize"]) \
  .option("isolationLevel", pg_props["isolationLevel"]) \
  .mode("append") \
  .save()

# Observação: para garantir idempotência/upsert, prefira escrever em staging table e depois executar MERGE/INSERT ON CONFLICT no Postgres.

spark.stop()
'''

### Notas e dicas práticas

* 'num_partitions' deve ser calibrado: mais partições = mais paralelismo, mas aumenta overhead por task. Para 20M, 100–400 dependendo de executors.
* 'batchsize': 5k–20k é comum; teste para achar o melhor no seu banco (10k é bom ponto inicial).
* Use 'repartition(<n>, chave_de_join)' se planeja fazer joins no destino por essa chave — evita shuffle extra depois.
* Para **upsert** (manter histórico ou atualizar registros):

  1. Grave em tabela 'staging' no Postgres (append).
  2. Execute no Postgres um 'MERGE' (Postgres 15) ou 'INSERT ... ON CONFLICT' combinando chaves, em uma transação. Isso é rápido dentro do banco.
* Se o PostgreSQL estiver no mesmo datacenter, avalie usar 'COPY' (mais rápido que JDBC). Para isso, exporte partições para CSV locais/temporários e faça 'COPY' via 'psycopg2'. Requer orquestração extra e acesso de rede/arquivo.
* Monitore I/O e conexões: muitos writers simultâneos no Postgres podem sobrecarregar o banco; limite 'repartition' se necessário.

---

# 2) Integrando Airflow com Spark (cluster standalone) — orquestração de ETL

Objetivo: usar Airflow para submeter jobs Spark a um cluster standalone e controlar dependências, retries e notificação.

### Arquitetura simples

* Airflow (scheduler + webserver + workers) executa 'SparkSubmitOperator' apontando para 'spark-submit' no master 'spark://master:7077'.
* Os artefatos ('job.py', jars, dependências') podem estar em um storage compartilhado (NFS, MinIO/S3 + download no job) ou em um repo/versionamento.

### Requisitos

* Airflow instalado com provider 'apache-airflow-providers-apache-spark'.
* 'spark-submit' e variáveis de ambiente acessíveis ao worker (ou use 'SparkSubmitOperator' com conexão que chame o master via remote).
* Drivers JDBC no classpath ou passados com '--jars'.

### Exemplo de DAG Airflow (arquivo 'dags/spark_etl_dag.py')

'''python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.operators.bash import BashOperator
from airflow.sensors.base import BaseSensorOperator

default_args = {
    "owner": "data-team",
    "depends_on_past": False,
    "email_on_failure": True,
    "retries": 2,
    "retry_delay": timedelta(minutes=5)
}

with DAG(
    dag_id="hana_to_postgres_etl",
    default_args=default_args,
    start_date=datetime(2025, 1, 1),
    schedule_interval="@daily",
    catchup=False,
    max_active_runs=1
) as dag:

    # Optional: sensor to ensure Spark master is reachable (simplified)
    check_spark = BashOperator(
        task_id="check_spark_master",
        bash_command="curl -sS http://spark-master:8080 || exit 1"
    )

    spark_job = SparkSubmitOperator(
        task_id="run_spark_hana_to_pg",
        application="/opt/jobs/job_hana_to_pg.py",  # onde seu script está
        conn_id="spark_default",  # configure conn in Airflow UI (Master URL)
        application_args=["--env", "prod"],
        total_executor_cores=16,
        executor_cores=4,
        executor_memory="8g",
        driver_memory="4g",
        jars="/opt/jars/hana-jdbc.jar,/opt/jars/postgres-jdbc.jar",
        spark_binary="spark-submit",  # opcional
        verbose=True
    )

    # Optional post-step: run SQL merge on Postgres to make upsert idempotent
    run_merge = BashOperator(
        task_id="run_postgres_merge",
        bash_command="psql postgresql://PG_USER:PG_PASS@pg-host:5432/dbname -f /opt/sql/merge_staging.sql"
    )

    check_spark >> spark_job >> run_merge
'''

### Boas práticas de orquestração

* Use 'max_active_runs=1' para evitar sobreposição de execuções quando job intenso.
* Parametrize DAG com data de execução e passe '--conf spark.some=...' se quiser controlar de runtime.
* Use 'XCom' para passar metadados (número de partições calculado dinamicamente, por exemplo).
* Garanta que 'jars' (drivers) estejam disponíveis; preferível centralizar jars em um path acessível por todos os workers.
* Trate falhas com retries e backoff; em caso de falhas recorrentes, registre logs e alerte por e-mail/Slack.
* Se precisar de mais controle (submit remoto, monitoramento de aplicação Spark), considere integrar com Livy para submeter e monitorar aplicações via REST.

---

# 3) Exemplo de fluxo NiFi: extrair de APIs REST e persistir em MinIO

Objetivo: desenhar um fluxo NiFi para consumir paginated REST APIs, transformar o JSON e gravar objetos em MinIO (S3 API compatível).

### Componentes principais do fluxo

1. **InvokeHTTP** — chamar a API REST (suporta autenticação, headers).
2. **EvaluateJsonPath / JoltTransformJSON** — extrair/transformar payload.
3. **SplitJson** — caso receba arrays e precise processar item a item.
4. **PutS3Object** (ou PutObject com S3 config) — gravar no MinIO (bucket).
5. **MergeContent** (opcional) — juntar pequenos eventos em lotes antes de gravar.
6. **RouteOnAttribute / Retry** — tratamento de erros e re-tentativa.
7. **PutFile** (opcional) — gravação local para fallback.
8. **Provenance + Bulletin** — observabilidade.

### Exemplo de fluxo (passo a passo)

1. **InvokeHTTP (get page 1)**

   * Método: GET
   * URL: 'https://api.exemplo.com/data?page=1'
   * Headers: Authorization Bearer, Accept: application/json
   * Configure 'Retry' behavior e 'ConnectTimeout'/'ReadTimeout'.

2. **EvaluateJsonPath**

   * Extrair 'next_page' e 'items' (ex.: '$.next', '$.data.items') para atributos de FlowFile.

3. **SplitJson** (se 'items' for array)

   * JsonPath Expression: '$.data.items[*]' to split para cada item, se quiser processar item a item.

4. **Optional: JoltTransformJSON**

   * Normalizar campos, remover aninhamentos, mapear nomes.

5. **UpdateAttribute**

   * Criar nome do arquivo final: ex.: 'filename = data_${epoch}_${uuid}.json'
   * Criar metadata úteis: 'object_key=folder/year=2025/month=09/${filename}'

6. **PutS3Object (MinIO)**

   * Endpoint: 'http://minio-host:9000'
   * Bucket: 'raw-api-data'
   * Credentials: accessKey/secretKey (configure controller service: 'S3CredentialsProvider')
   * Object Key: '${object_key}'
   * Content-Type: 'application/json'
   * Configure 'Storage Class' se necessário.

7. **On Success / Failure**

   * Success -> 'success' relationship: enviar para 'LogAttribute' ou tópico Kafka (se existir) para downstream.
   * Failure -> 'retry' relationship: route para 'RetryFlow' que usa 'Wait/Notify' ou 'RetryFlowFile' com backoff; após n tentativas mover para 'dead-letter' (PutFile for manual review).

8. **Paginacao automatica**

   * Use um loop com 'InvokeHTTP' que pega 'next_page' de 'EvaluateJsonPath' e re-inserir para 'InvokeHTTP' até 'next_page' ser nulo. Ou use 'GenerateFlowFile' + 'InvokeHTTP' em combinação com 'ExtractText' para iterar.

### Atenção a performance e pequenos arquivos

* Se a API retorna muitos pequenos itens, use 'MergeContent' para agrupar N itens por arquivo antes de enviar ao MinIO (reduz número de objetos pequenos).
* Configure 'PutS3Object' com 'Multipart Upload' habilitado se arquivos maiores; caso contrário ajuste lote para evitar muitos objetos pequenos.
* Controle de concorrência em NiFi (concurrent tasks) para não estourar a API (rate-limits) nem sobrecarregar MinIO.

### Esquema visual (simples)

'''
[GenerateFlowFile / Cron] -> [InvokeHTTP] -> [EvaluateJsonPath] -> [SplitJson] -> [JoltTransformJSON] -> [MergeContent (opcional)] -> [PutS3Object (MinIO)]
                                                      \-> onFailure -> [RouteOnAttribute] -> [Retry or DeadLetter]
'''

---

## Dicas finais e checklist rápido

* **Teste de carga**: faça testes com 20% do volume e monitore I/O, CPU, GC e conexões de DB antes de rodar full.
* **Observabilidade**: habilite métricas e logs (Spark UI, Airflow logs, NiFi provenance, MinIO metrics).
* **Idempotência**: sempre que possível, grave em staging e faça idempotent merge/upsert no destino.
* **Backpressure e rate limits**: throttle quando consumir APIs externas.
* **Segurança**: mantenha credenciais em vault/SecretManager; use TLS nas conexões JDBC e S3.

O Airflow tem um agendador nativo, chamado Scheduler, que é justamente o coração da orquestração. Ele fica responsável por:

Ler as DAGs (arquivos Python que você define).

Interpretar os parâmetros de agendamento (schedule_interval, start_date, end_date, catchup, etc.).

Decidir quando cada DAG deve rodar.

Colocar as tasks na fila para serem executadas pelos workers (Celery, Kubernetes, Local Executor, etc.).

Como funciona na prática

Cada DAG pode ter um agendamento do tipo cron ("0 2 * * *" → todos os dias às 2h).

Também pode ser @daily, @hourly, @once, None (apenas execução manual).

O scheduler verifica periodicamente os DAGs e dispara as execuções quando chega a hora.

Exemplo
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator

default_args = {
    "owner": "francisco",
    "retries": 1,
    "retry_delay": timedelta(minutes=5)
}

with DAG(
    dag_id="exemplo_scheduler",
    default_args=default_args,
    start_date=datetime(2025, 9, 1),
    schedule_interval="@daily",  # executa todo dia à meia-noite
    catchup=False
) as dag:

    tarefa = BashOperator(
        task_id="hello",
        bash_command="echo 'Rodando DAG via scheduler!'"
    )


Se você subir o Airflow com webserver, scheduler e worker, esse DAG rodará todos os dias automaticamente.
