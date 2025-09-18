
# O que é o Apache XTable?

O **Apache XTable** é uma ferramenta de código aberto, atualmente em incubação na Apache Software Foundation, que atua como um **conversor entre formatos de tabelas de _lakehouse_ de dados**.

Ele não cria um novo formato de tabela, mas sim uma camada de tradução que possibilita a interoperabilidade entre os formatos de tabela de código aberto mais populares, como **Apache Hudi**, **Apache Iceberg** e **Delta Lake**.

---

### Como ele funciona?

O Apache XTable™ lê os metadados existentes da sua tabela e grava metadados para um ou mais formatos de tabela, aproveitando as APIs existentes fornecidas por cada projeto de formato de tabela. Os metadados serão persistidos em um diretório no caminho base da sua tabela (_delta_log para Delta, metadata para Iceberg e .hoodie para Hudi). Isso permite que seus dados existentes sejam lidos como se tivessem sido gravados usando Delta, Hudi ou Iceberg. Por exemplo, um leitor Spark pode usar spark.read.format(“delta | hudi | iceberg>”).load(“path/to/data”). 

Em um **_data lakehouse_**, os dados são geralmente armazenados em arquivos como Parquet. A forma como o controle de versão, o esquema e as transações são gerenciados é definida pelo formato da tabela. O Apache XTable lê os metadados de um formato (por exemplo, Hudi) e, a partir deles, cria e sincroniza os metadados para outro formato (como Iceberg ou Delta Lake) no mesmo conjunto de arquivos.

Isso significa que uma tabela escrita em um formato pode ser lida por mecanismos de consulta que suportam os outros formatos, **sem a necessidade de duplicar ou reescrever os dados**.

---

### Principais Características

* **Interoperabilidade Omnidirecional**: O XTable permite a conversão e sincronização de metadados em qualquer direção entre os formatos suportados (Hudi, Iceberg, Delta Lake).
* **Sincronização de Metadados**: Ele sincroniza não apenas os arquivos de dados, mas também estatísticas de colunas, metadados de partição e atualizações de esquema, garantindo consistência entre os formatos.
* **Escolha Flexível**: Com o XTable, equipes de dados podem escolher o formato de tabela que melhor se adapta às suas ferramentas e necessidades. Por exemplo, uma equipe pode usar Databricks (com suporte nativo a Delta Lake) enquanto outra usa Snowflake (com suporte a Iceberg) para acessar a mesma tabela.
* **Modos de Sincronização**: Oferece dois modos principais:
    * **Sincronização Incremental**: Um modo leve e de alto desempenho, ideal para tabelas grandes.
    * **Sincronização Completa**: Utilizado quando a sincronização incremental não é aplicável.

 ---
 
### Quando devo usar ?
Ele será multo util nos cenários onde você precise usar o que as melhores futures de cada ferramenta, por exemplo ingestão ou indexação extremamente rápidas do Hudi e tabela gerenciadas do Hudi, acelerações de consultas por fótons do Delta Lake dentro do Databricks. Em fim Independentemente da combinação de formatos que você precisar, o Apache XTable vai garantir que você possa se beneficiar dos formatos que hoje dominam o mercado. 
