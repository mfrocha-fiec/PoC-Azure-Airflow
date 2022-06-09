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

### Usando o SDK com Airflow no AKS:
