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
Foi criada uma cópia do Airflow do Chico 3.0 com uma dag "teste_azure".

O fluxo consiste em criar a task em um arquivo .py e salvar no ADLS Gen 2.

A task do Python chamará um script helper para executar um batch job spark usando o cluster synapse. O arquivo .py será passado no argumento dessa função.

O primeiro passo é a criação de um registro para Apps on premises na azure:
https://docs.microsoft.com/en-us/azure/developer/python/sdk/authentication-on-premises-apps?tabs=azure-portal.

Isso gerará um ```client id```, ```tenant id``` e ```client secret```.

É necessário instalar os pacotes ```azure-identity``` e ```azure-synapse```.

Com isso, pode-se usar o tutorial [neste link](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/synapse/azure-synapse/samples/sample.py) para criar uma classe que lidará com a autenticação, listar, criar e deletar Jobs no ASASP.

Para submeter um Job, precisamos de um arquivo de definição do Job, que é o script .py que executará a tarefa. Um dos problemas é que esse arquivo de definição *precisa* estar no Blob Storage. O problema é que fica mais difícil versionar esse arquivo.

![image](https://user-images.githubusercontent.com/83727621/172836606-2c1d2c61-485e-43e6-b80f-4116773e6a76.png)

Uma das possíveis soluções é criar um gatilho que copiará o .py do local onde ele está no git e subirá no Blob Storage no momento da execução. Uma ideia de como fazer isso foi discutida [nessa thread](https://stackoverflow.com/questions/68234041/azure-devops-ci-cd-pipelines-for-adls-gen2-resource).

![image](https://user-images.githubusercontent.com/83727621/172838382-e0312384-501a-4daa-877f-abd1eb044f55.png)

O primeiro script da ETL de teste é o ```extract.py``` que vai baixar o arquivo .csv da rede e salvar no ADLS Gen 2. Para isso é necessário ter uma ```Azure Storage connection string```, obtida através [desse tutorial](https://docs.microsoft.com/en-us/azure/storage/common/storage-configure-connection-string).

### Usando o SDK com Airflow no AKS:
