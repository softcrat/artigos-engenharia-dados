## O que é o Apache Airflow?

O Apache Airflow é uma plataforma open source criada pelo Airbnb em 2014 e atualmente mantida pela Apache Software Foundation.
Ele foi desenvolvido para orquestrar fluxos de trabalho (workflows) de forma programática, permitindo que engenheiros de dados e times de analytics possam agendar, monitorar e gerenciar pipelines de dados complexos.

Com o Airflow, ao invés de configurar processos manualmente, você define workflows como código (o famoso DAG – Directed Acyclic Graph), garantindo maior reprodutibilidade, escalabilidade e controle.

Principais Conceitos

DAG (Directed Acyclic Graph)

É a estrutura que define o workflow.

Cada DAG é composto por um conjunto de tarefas com dependências entre si, garantindo que sejam executadas em uma ordem lógica.

Tarefas (Tasks)

São as unidades de trabalho dentro do DAG.

Podem representar, por exemplo: leitura de dados, transformação em Spark, carga em um banco, envio de relatórios.

Operadores (Operators)

São "templates" de tarefas que definem o que será feito. Exemplos:

BashOperator – executa comandos no shell.

PythonOperator – executa funções Python.

PostgresOperator – executa queries SQL em bancos PostgreSQL.

SparkSubmitOperator – submete jobs Spark.

Scheduler e Executor

O Scheduler agenda os DAGs conforme definido.

O Executor executa as tarefas, podendo ser local (LocalExecutor), distribuído (CeleryExecutor) ou em Kubernetes.

Web UI

Uma interface visual para acompanhar execução, logs, status de tarefas e reexecutar processos quando necessário.

Por que usar o Airflow?

Escalabilidade: suporta desde pipelines simples até orquestração de milhares de jobs distribuídos.

Flexibilidade: por ser Python-based, permite integração com praticamente qualquer ferramenta.

Observabilidade: logs, métricas e alertas ajudam a monitorar cada etapa do pipeline.

Comunidade ativa: grande ecossistema de operadores e integrações já prontos.

Exemplos de Uso

Orquestrar um pipeline ETL:

Extrair dados de uma API.

Transformar no Spark.

Salvar em um Data Warehouse.

Agendar tarefas recorrentes:

Geração de relatórios diários em CSV.

Atualização de dashboards no Power BI.

Controlar dependências:

Só carregar dados na camada ouro após validar a qualidade na camada prata.

Exemplo Simples de DAG em Airflow
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

# Definição do DAG
with DAG(
    dag_id="exemplo_airflow",
    start_date=datetime(2025, 1, 1),
    schedule_interval="@daily",
    catchup=False,
) as dag:

    tarefa1 = BashOperator(
        task_id="extrair_dados",
        bash_command="echo 'Extraindo dados...'"
    )

    tarefa2 = BashOperator(
        task_id="transformar_dados",
        bash_command="echo 'Transformando dados...'"
    )

    tarefa3 = BashOperator(
        task_id="carregar_dados",
        bash_command="echo 'Carregando dados no DW...'"
    )

    # Definindo ordem de execução
    tarefa1 >> tarefa2 >> tarefa3


Neste exemplo, o pipeline roda diariamente e executa três etapas em sequência: extração → transformação → carga.

Quando usar o Apache Airflow?

Quando há necessidade de agendamento e monitoramento de pipelines complexos.

Para substituir scripts manuais ou cron jobs difíceis de manter.

Quando se deseja integrar diferentes sistemas (APIs, bancos de dados, cloud, Spark etc.) em uma mesma orquestração.

Conclusão

O Apache Airflow é hoje uma das ferramentas mais utilizadas em engenharia de dados para orquestrar pipelines de forma escalável e confiável. Ele resolve problemas comuns de automação, integração e controle de processos de dados.
