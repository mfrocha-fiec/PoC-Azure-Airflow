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

É feito o download de um .csv no github para um DataFrame Pandas.



### Limitações 

Para submeter um Job, precisamos de um arquivo de definição do Job, que é o script .py que executará a tarefa. Um dos problemas é que esse arquivo de definição *precisa* estar no Blob Storage. O problema é que fica mais difícil versionar esse arquivo.

Uma das possíveis soluções é criar um gatilho que copiará o .py do local onde ele está no git e subirá no Blob Storage no momento da execução. Uma ideia de como fazer isso foi discutida [nessa thread](https://stackoverflow.com/questions/68234041/azure-devops-ci-cd-pipelines-for-adls-gen2-resource).

![image](https://user-images.githubusercontent.com/83727621/172838382-e0312384-501a-4daa-877f-abd1eb044f55.png)

### Usando o SDK com Airflow no AKS:
