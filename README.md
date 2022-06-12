# PoC-Azure-Airflow
Repositório para documentar o PoC de um ambiente Azure integrado com Airflow On Premises

Temos três possíveis opções de ambiente:
- Azure Synapse Analytics Spark Pool (ASASP) [preferido];
- Azure HDInsights;
- Azure Databricks.

## Opção 1: Azure Synapse Analytics Spark Pool (ASASP)
O plano inicial é usar o Azure SDK Python para disparar as tasks no ASASP.

Há um [tutorial](https://docs.microsoft.com/en-us/azure/synapse-analytics/spark/vscode-tool-synapse#open-a-work-folder) de como attachar o cluster no VSCode para executar códigos diretamente no cluster remoto. 

A extensão foi descontinuada. 

### Usando o SDK com Airflow local:

O primeiro passo é a criação de um registro para Apps on premises na azure:
https://docs.microsoft.com/en-us/azure/developer/python/sdk/authentication-on-premises-apps?tabs=azure-portal.

Isso gerará um ```client id```, ```tenant id``` e ```client secret```. O app gerado desse registro vira uma espécie de usuário no AD. É necessário dar a esse usuário as permissões para poder interagir com os recursos.

A solução paliativa encontrada para evitar problemas de permissão é adicionar o app como um Owner da assinatura como um todo. O ideal é adicionar as permissões granulares pro app apenas do que ele está responsável por administrar.

![image](https://user-images.githubusercontent.com/83727621/173186302-7942fefc-bc4b-43ba-bb5e-5ce1f90f613c.png)

### Configuração do ambiente airflow

Foi criada uma cópia do Airflow do Chico 3.0 com uma dag "teste_azure".
É necessário instalar os pacotes ```azure-identity```, ```azure-synapse```, ```adlfs``` e a extensão do Airflow ```apache-airflow-providers-microsoft-azure```.

### Fluxo de mockup

O fluxo consiste em baixar um arquivo .csv da internet, transformar ele em .parquet no Spark Pool e salvar na camada RAW.

### Extração

É feito o download de um .csv no github em memória. É feito o upload desse arquivo para o Storage Account, no container ```landing```.

```
def extract(dag_name):
    storage_account_name = 'pocairflow'

    initialize_storage_account_ad(storage_account_name)
    
    url = r'https://code.datasciencedojo.com/datasciencedojo/datasets/raw/master/Accidental%20Drug%20Related%20Deaths%20in%20Connecticut,%20US/Accidental%20Drug%20Related%20Deaths%20in%20Connecticut-2012-2018.csv'

    file_system = 'landing'
    path_directory=f'/{dag_name}'

    request = requests.get(url)

    file_system_client = service_client.get_file_system_client(file_system=file_system)

    directory_client = get_directory_client_or_create(file_system_client, path_directory)    
    
    file_client = directory_client.create_file("drug-data-2012-2018.csv")

    file_contents = request.content

    file_client.append_data(data=file_contents, offset=0, length=len(file_contents))
    file_client.flush_data(len(file_contents))
```

### Transformação

A transformação é bem simples, apenas lê o .csv com PySpark e salva na camada ```raw``` como Parquet.

transform_raw.py:
```
import sys
 
from pyspark import SparkContext, SparkConf
from pyspark.sql import SparkSession

if __name__ == "__main__":
	
	# create Spark context with necessary configuration
	conf = SparkConf().setAppName("transform-raw-nb1").set("spark.hadoop.validateOutputSpecs", "false")
	spark = SparkSession.builder.config(conf=conf) \
        .appName('so')\
        .getOrCreate()	

	source_path = "abfss://landing@pocairflow.dfs.core.windows.net/teste_azure/drug-data-2012-2018.csv"
	target_path = "abfss://raw@pocairflow.dfs.core.windows.net/teste_azure/drug-data-2012-2018.parquet"

	df = spark.read.option("header","true").csv(source_path) 

	df.write.format("parquet").mode("overwrite").save(target_path)
```
Foi criado um wrapper pra encapsular o método da Azure para submeter o Job Spark, tornando mais harmônico com o fluxo do Airflow.

synapse_tools.py
```
from azure.synapse.spark import SparkClient
from azure.synapse.spark.models import SparkBatchJobOptions, SparkSessionOptions, SparkStatementOptions
from azure.identity import DefaultAzureCredential
from time import sleep

def submit_spark_job_synapse(options_dict:dict):
    token_credential = DefaultAzureCredential()
    endpoint = 'https://pocairflow.dev.azuresynapse.net'

    spark_pool_name = 'pocairflow'

    spark_client = SparkClient(token_credential, endpoint, spark_pool_name)

    options = SparkBatchJobOptions.from_dict(options_dict)

    job = spark_client.spark_batch.create_spark_batch_job(options, detailed=True)
    job_id = job.id

    not_finished = True

    while not_finished:
        job_update = spark_client.spark_batch.get_spark_batch_job(job_id)
        if job_update.state in ['running', 'not_started']:
            sleep(30)
            pass
        elif job_update.state == 'success':
            not_finished = False
        elif job_update.state == 'failed':
            raise "The job runned with problems. Check in Azure Portal."
        else:
            sleep(30)
```
Essa função pega as credenciais do App Azure diretamente das variáveis de ambiente, sendo elas: ```AZURE_SUBSCRIPTION_ID```, ```AZURE_TENANT_ID```, ```AZURE_CLIENT_ID``` e ```AZURE_CLIENT_SECRET```.

A função cria o Job no cluster, sendo configurado através do input ```options_dict```. Um exemplo de como é esse dict:

```
options_list = {
    "tags": None,
    "artifactId": None,
    "name": f"transform_raw",
    "file": f'abfss://scripts@pocairflow.dfs.core.windows.net/{dag_name}/transform_raw.py',
    "className": None,
    "args": None,
    "jars": [],
    "files": [],
    "archives": [],
    "conf": None,
    "driverMemory": "4g",
    "driverCores": 4,
    "executorMemory": "2g",
    "executorCores": 2,
    "numExecutors": 2,
}
```
Onde o argumento ```file``` se refere ao script .py da transformação que deve estar no Storage, sendo ele Gen1 ou Gen2.


### Limitações 

Para submeter um Job, precisamos de um arquivo de definição do Job, que é o script .py que executará a tarefa. Um dos problemas é que esse arquivo de definição *precisa* estar no Blob Storage. O problema é que fica mais difícil versionar esse arquivo.

Uma das possíveis soluções é criar um gatilho que copiará o .py do local onde ele está no git e subirá no Blob Storage no momento da execução. Uma ideia de como fazer isso foi discutida [nessa thread](https://stackoverflow.com/questions/68234041/azure-devops-ci-cd-pipelines-for-adls-gen2-resource).

![image](https://user-images.githubusercontent.com/83727621/172838382-e0312384-501a-4daa-877f-abd1eb044f55.png)

### Usando o SDK com Airflow no AKS:
