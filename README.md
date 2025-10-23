# Processamento de fluxo com Azure Databricks

Esta arquitetura de referência mostra um pipeline de [processamento de fluxo](/azure/architecture/data-guide/big-data/real-time-processing) e ponta a ponta. Esse tipo de pipeline possui quatro estágios: ingestão, processamento, armazenamento e análise e geração de relatórios.
Para esta arquitetura de referência, o pipeline ingere dados de duas fontes, realiza uma junção entre registros relacionados de cada fluxo, enriquece o resultado e calcula uma média em tempo real. Os resultados são armazenados para análises posteriores.

![](https://github.com/mspnp/architecture-center/blob/master/docs/reference-architectures/data/images/stream-processing-databricks.png)

**Cenário:** uma empresa de táxi coleta dados sobre cada viagem. Para este cenário, assumimos que existem dois dispositivos distintos enviando dados. O táxi possui um taxímetro que envia informações sobre cada corrida — duração, distância e locais de embarque e desembarque. Um dispositivo separado aceita pagamentos dos clientes e envia dados sobre as tarifas.
Para identificar tendências de uso, a empresa de táxi deseja calcular a gorjeta média por milha percorrida, em tempo real, para cada bairro.

## Implante a solução

Uma implantação para esta arquitetura de referência está disponível no [GitHub](https://github.com/mspnp/azure-databricks-streaming-analytics).

### Pré-requisitos

1. Clone, faça um fork ou baixe este repositório GitHub.

2. Instale o [Docker](https://www.docker.com/) para executar o gerador de dados.

3. Instale o [Azure CLI 2.0](/cli/azure/install-azure-cli?view=azure-cli-latest).

4. Instale o [Databricks CLI](https://docs.databricks.com/user-guide/dev-tools/databricks-cli.html).

5. Em um prompt de comando, prompt bash ou prompt do PowerShell, faça login na sua conta do Azure da seguinte maneira:
```shell
az login
```
6. Instale um IDE Java com os seguintes recursos:
- JDK 1.8
- Scala SDK 2.11
- Maven 3.5.4

### Baixe os arquivos de dados sobre táxis e bairros da cidade de Nova York

1. Crie um diretório chamado `DataFile` na raiz do repositório Github clonado em seu sistema de arquivos local.

2. Abra um navegador da web e acesse https://uofi.app.box.com/v/NYCtaxidata/folder/2332219935.

3. Clique no botão **Download** nesta página para baixar um arquivo zip com todos os dados de táxi daquele ano.

4. Extraia o arquivo zip para o diretório `DataFile`.

    > [!NOTA]
    > Este arquivo zip contém outros arquivos zip. Não extraia os arquivos zip filhos.

    A estrutura do diretório deve ficar assim:

```shell
    /DataFile
        /FOIL2013
            trip_data_1.zip
            trip_data_2.zip
            trip_data_3.zip
            ...
    ```
5. Abra um navegador da Web e acesse https://www.zillow.com/howto/api/neighborhood-boundaries.htm. 

6. Clique em **New York Neighborhood Boundaries** (Limites dos bairros de Nova York) para baixar o arquivo.

7. Copie o arquivo **ZillowNeighborhoods-NY.zip** do diretório **downloads** do seu navegador para o diretório `DataFile`.

### Implante os recursos do Azure

1. Em um shell ou no Prompt de Comando do Windows, execute o comando a seguir e siga as instruções de login:

```bash
    az login
    ```

2. Navegue até a pasta chamada `azure` no repositório GitHub:

```bash
    cd azure
    ```

3. Execute os seguintes comandos para implantar os recursos do Azure:

```bash
    export resourceGroup=‘[Nome do grupo de recursos]’
    export resourceLocation=‘[Região]’
    export eventHubNamespace=‘[Nome do namespace do Event Hubs]’
    export databricksWorkspaceName=‘[Nome do espaço de trabalho do Azure Databricks]’
    export cosmosDatabaseAccount='[Nome do banco de dados do Cosmos DB]'
    export logAnalyticsWorkspaceName=‘[Nome do espaço de trabalho do Log Analytics]’
    export logAnalyticsWorkspaceRegion=‘[Região do Log Analytics]’

    # Crie um grupo de recursos
    az group create --name $resourceGroup --location $resourceLocation

    # Implante recursos
    az group deployment create --resource-group $resourceGroup \
        --template-file deployresources.json --parameters \
        eventHubNamespace=$eventHubNamespace \
        databricksWorkspaceName=$databricksWorkspaceName \
        cosmosDatabaseAccount=$cosmosDatabaseAccount \
	    logAnalyticsWorkspaceName=$logAnalyticsWorkspaceName \
        logAnalyticsWorkspaceRegion=$logAnalyticsWorkspaceRegion
    ```

4. A saída da implantação é gravada no console após a conclusão. Pesquise a seguinte JSON na saída:

```JSON
“outputs”: {
        “cosmosDb”: {
          “type”: “Object”,
          “value”: {
            “hostName”: <value>,
            “secret”: <value>,
            “username”: <value>
          }
        },
        “eventHubs”: {
          “type”: “Object”,
          “value”: {
            “taxi-fare-eh”: <valor>,
            “taxi-ride-eh”: <valor>
          }
        },
        “logAnalytics”: {
          “tipo”: “Objeto”,
          “valor”: {
            “segredo”: <valor>,
            “workspaceId”: <valor>
          }
        }
},
```
Esses valores são os segredos que serão adicionados aos segredos do Databricks nas próximas seções. Mantenha-os em segurança até adicioná-los nessas seções.

### Adicionar uma tabela Cassandra à conta do Cosmos DB

1. No portal do Azure, navegue até o grupo de recursos criado na seção **implantar os recursos do Azure** acima. Clique em **Conta do Azure Cosmos DB**. Crie uma tabela com a API Cassandra.

2. Na guia **Visão geral**, clique em **Adicionar tabela**.

3. Quando a guia **Adicionar tabela** for aberta, digite `newyorktaxi` na caixa de texto **Nome do espaço de chaves**. 

4. Na seção **Digite o comando CQL para criar a tabela**, digite `neighborhoodstats` na caixa de texto ao lado de `newyorktaxi`.

5. Na caixa de texto abaixo, digite o seguinte:
```shell
(neighborhood text, window_end timestamp, number_of_rides bigint,total_fare_amount double, primary key(neighborhood, window_end))
```
6. Na caixa de texto **Throughput (1.000 - 1.000.000 RU/s)**, insira o valor `4000`.

7. Clique em **OK**.

### Adicione os segredos do Databricks usando a CLI do Databricks

Primeiro, insira os segredos para o EventHub:

1. Usando a **CLI do Azure Databricks** instalada na etapa 2 dos pré-requisitos, crie o escopo secreto do Azure Databricks:
```shell
    databricks secrets create-scope --scope “azure-databricks-job”
    ```
2. Adicione o segredo para o EventHub de corrida de táxi:
```shell
    databricks secrets put --scope “azure-databricks-job” --key “taxi-ride”
    ```
    Depois de executado, esse comando abre o editor vi. Insira o valor **taxi-ride-eh** da seção de saída **eventHubs** na etapa 4 da seção *implantar os recursos do Azure*. Salve e saia do vi.

3. Adicione o segredo para o EventHub da tarifa de táxi:
```shell
    databricks secrets put --scope “azure-databricks-job” --key “taxi-fare”
    ```
    Depois de executado, esse comando abre o editor vi. Digite o valor **taxi-fare-eh** da seção de saída **eventHubs** na etapa 4 da seção *implantar os recursos do Azure*. Salve e saia do vi.

Em seguida, digite os segredos para o Cosmos DB:

1. Abra o portal do Azure e navegue até o grupo de recursos especificado na etapa 3 da seção **implantar os recursos do Azure**. Clique na conta do Azure Cosmos DB.

2. Usando a **CLI do Azure Databricks**, adicione o segredo para o nome de usuário do Cosmos DB:
```shell
    databricks secrets put --scope azure-databricks-job --key “cassandra-username”
    ```
Depois de executado, esse comando abre o editor vi. Insira o valor do **nome de usuário** da seção de saída **CosmosDb** na etapa 4 da seção *implantar os recursos do Azure*. Salve e saia do vi.

3. Em seguida, adicione o segredo para a senha do Cosmos DB:
```shell
    databricks secrets put --scope azure-databricks-job --key “cassandra-password”
Depois de executado, esse comando abre o editor vi. Digite o valor **secret** da seção de saída **CosmosDb** na etapa 4 da seção *implantar os recursos do Azure*. Salve e saia do vi.

> [!NOTA]
> Se estiver usando um [escopo secreto com suporte do Azure Key Vault](https://docs.azuredatabricks.net/user-guide/secrets/secret-scopes.html#azure-key-vault-backed-scopes), o escopo deve ser nomeado **azure-databricks-job** e os segredos devem ter exatamente os mesmos nomes que os acima.

### Adicione o arquivo de dados Zillow Neighborhoods ao sistema de arquivos Databricks

1. Crie um diretório no sistema de arquivos Databricks:
```bash
dbfs mkdirs dbfs:/azure-databricks-jobs
```

2. Navegue até o diretório `DataFile` e digite o seguinte:
```bash
    dbfs cp ZillowNeighborhoods-NY.zip dbfs:/azure-databricks-jobs
    ```

### Adicione o ID da área de trabalho do Azure Log Analytics e a chave primária aos arquivos de configuração

Para esta seção, você precisa do ID da área de trabalho do Log Analytics e da chave primária. O ID da área de trabalho é o valor **workspaceId** da seção de saída **logAnalytics** na etapa 4 da seção *implantar os recursos do Azure*. A chave primária é o **secret** da seção de saída. 

1. Para configurar o registro log4j, abra `\azure\AzureDataBricksJob\src\main\resources\com\microsoft\pnp\azuredatabricksjob\log4j.properties`. Edite os dois valores a seguir:
```shell
    log4j.appender.A1.workspaceId=<ID da área de trabalho do Log Analytics>
    log4j.appender.A1.secret=<Chave primária do Log Analytics>
```
2. Para configurar o registro personalizado, abra `\azure\azure-databricks-monitoring\scripts\metrics.properties`. Edite os dois valores a seguir:
```shell
    *.sink.loganalytics.workspaceId=<ID do espaço de trabalho do Log Analytics>
    *.sink.loganalytics.secret=<Chave primária do Log Analytics>
    ```

### Crie os arquivos .jar para a tarefa do Databricks e o monitoramento do Databricks

1. Use seu IDE Java para importar o arquivo de projeto Maven chamado **pom.xml** localizado no diretório raiz. 

2. Execute uma compilação limpa. O resultado dessa compilação são os arquivos chamados **azure-databricks-job-1.0-SNAPSHOT.jar** e **azure-databricks-monitoring-0.9.jar**. 

### Configure o registro personalizado para a tarefa do Databricks

1. Copie o arquivo **azure-databricks-monitoring-0.9.jar** para o sistema de arquivos do Databricks inserindo o seguinte comando na **CLI do Databricks**:
    ```shell
    databricks fs cp --overwrite azure-databricks-monitoring-0.9.jar dbfs:/azure-databricks-job/azure-databricks-monitoring-0.9.jar
    ```

2. Copie as propriedades de registro personalizadas de `\azure\azure-databricks-monitoring\scripts\metrics.properties` para o sistema de arquivos Databricks inserindo o seguinte comando:
```shell
    databricks fs cp --overwrite metrics.properties dbfs:/azure-databricks-job/metrics.properties
    ```

3. Se você ainda não decidiu um nome para o seu cluster Databricks, selecione um agora. Você inserirá o nome abaixo no caminho do sistema de arquivos Databricks para o seu cluster. Copie o script de inicialização de `\azure\azure-databricks-monitoring\scripts\spark.metrics` para o sistema de arquivos Databricks inserindo o seguinte comando
```
    databricks fs cp --overwrite spark-metrics.sh dbfs:/databricks/init/<cluster-name>/spark-metrics.sh
    ```

### Crie um cluster Databricks
