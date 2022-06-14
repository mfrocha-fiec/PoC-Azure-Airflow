# PoC-Azure-Airflow
Repositório para documentar o PoC de um ambiente Azure integrado com Airflow On Premises

Temos três possíveis opções de ambiente:
- Azure Synapse Analytics Spark Pool (ASASP) [preferido];
- Azure HDInsights;
- Azure Databricks.

## 1. Opção 1: Azure Synapse Analytics Spark Pool (ASASP)
O plano inicial é usar o Azure SDK Python para disparar as tasks no ASASP.

### 1.1 Map da solução:

![fluxo_synapse drawio](https://user-images.githubusercontent.com/83727621/173453770-ddd58511-0817-4f5e-9e97-1afc908f064e.png)

### 1.2 Usando o SDK com Airflow local:

O primeiro passo é a criação de um registro para Apps on premises na azure:
https://docs.microsoft.com/en-us/azure/developer/python/sdk/authentication-on-premises-apps?tabs=azure-portal.

Isso gerará um ```client id```, ```tenant id``` e ```client secret```. O app gerado desse registro vira uma espécie de usuário no AD. É necessário dar a esse usuário as permissões para poder interagir com os recursos.

A solução paliativa encontrada para evitar problemas de permissão é adicionar o app como um Owner da assinatura como um todo. O ideal é adicionar as permissões granulares pro app apenas do que ele está responsável por administrar.

![image](https://user-images.githubusercontent.com/83727621/173186302-7942fefc-bc4b-43ba-bb5e-5ce1f90f613c.png)

### 1.3 Configuração do ambiente airflow

Foi criada uma cópia do Airflow do Chico 3.0 com uma dag "teste_azure".
É necessário instalar os pacotes ```azure-identity```, ```azure-synapse```, ```adlfs``` e a extensão do Airflow ```apache-airflow-providers-microsoft-azure```.

### 1.4 Fluxo de mockup

O fluxo consiste em baixar um arquivo .csv da internet, transformar ele em .parquet no Spark Pool e salvar na camada RAW.

### 1.4.1 Extração

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

### 1.4.2 Transformação

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

A função também checa o estado da execução do Job no cluster, para retornar ao Airflow se houve erros ou não.

### 1.5 Limitações 

Para submeter um Job, precisamos de um arquivo de definição do Job, que é o script .py que executará a tarefa. Um dos problemas é que esse arquivo de definição *precisa* estar no Blob Storage. O problema é que fica mais difícil versionar esse arquivo.

Uma das possíveis soluções é criar um gatilho que copiará o .py do local onde ele está no git e subirá no Blob Storage no momento da execução. Uma ideia de como fazer isso foi discutida [nessa thread](https://stackoverflow.com/questions/68234041/azure-devops-ci-cd-pipelines-for-adls-gen2-resource).

![image](https://user-images.githubusercontent.com/83727621/172838382-e0312384-501a-4daa-877f-abd1eb044f55.png)

## 2. Opção 2: Azure Databricks

É feita a criação do cluster Databricks. A observação a ser feita é que não tinha opções de criar o cluster nas regiões mais comuns (EUA, Brazil).
As instãncias são de VMs da Azure, então tendem a ser caras (atualmente usando uma de U$ 250).

### 2.1 Usando o pacote ```dbx```

A primeira opção é utilizando o pacote ```dbx```, que de acordo com os tutoriais da azure, é o que será mantido de agora em diante. [Este tutorial](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/dbx#code-example) explica como criar um job básico utilizando essa ferramenta.

O tutorial é bem simples, mas de forma resumida, o pacote "builda" um projeto para o Databricks. Fica uma solução esquisita, já que é necessário ter vários diretórios e arquivos de configuração para rodar apenas um Job.

![image](https://user-images.githubusercontent.com/83727621/173466283-a52dd3a8-2e09-4216-bcc6-67e429075aa0.png)

A vantagem é que essa forma de deploy não requer que o script de execução esteja no Gen 2. A desvantagem é que dificulta a implantação em um fluxo de trabalho.

### 2.2 Usando os hooks e operators do Airflow (API)

O Airflow já conta com algumas integrações com o Databricks. O pacote ```apache-airflow-providers-databricks``` foi baseado na [API do Databricks](https://docs.databricks.com/dev-tools/api/index.html).

#### 2.2.1 Configurando conexão

A conexão na interface do Airflow terá os seguintes campos: 
![image](https://user-images.githubusercontent.com/83727621/173550561-c464716d-2dcf-411a-87ba-0c34a62b4a48.png)

O Host é obtido da URL que se acessa no studio Databricks.
A autenticação pode ser feita por AAD ou por PAT (Personal Access Token). O método mais fácil é com o PAT. Um tutorial de como obter o PAT no Databricks pode ser encontrado [aqui](https://docs.databricks.com/dev-tools/api/latest/authentication.html).

#### 2.2.2 Usando o DatabricksSubmitRunOperator

A extensão Databricks para Airflow conta com vários operadores para realizar diversas tasks administrativas no cluster. Este [link](https://airflow.apache.org/docs/apache-airflow-providers-databricks/stable/operators/index.html) contém uma lista completa do que cada operador faz.

O operador utilizado no PoC é o ```DatabricksSubmitRunOperator```, que permite executar uma task Spark através de diversos meios, incluindo um script ```.py```, o mais relevante no caso da utilização com o Airflow.




