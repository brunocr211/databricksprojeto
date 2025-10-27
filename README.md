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
1. No espaço de trabalho do Databricks, clique em “Clusters” e, em seguida, clique em “criar cluster”. Digite o nome do cluster que você criou na etapa 3 da seção **configurar registro personalizado para a tarefa do Databricks** acima.

2. Selecione um modo de cluster **padrão**.

3. Defina a **versão de tempo de execução do Databricks** como **4.3 (inclui Apache Spark 2.3.1, Scala 2.11)**

4. Defina a **versão do Python** como **2**.

5. Defina **Tipo de driver** como **O mesmo que o trabalhador**

6. Defina **Tipo de trabalhador** como **Standard_DS3_v2**.

7. Defina **Mínimo de trabalhadores** como **2**.

8. Desmarque **Habilitar autoescala**. 

9. Abaixo da caixa de diálogo **Encerramento automático**, clique em **Scripts de inicialização**. 

10. Digite **dbfs:/databricks/init/<cluster-name>/spark-metrics.sh**, substituindo o nome do cluster criado na etapa 1 por <cluster-name>.

11. Clique no botão **Adicionar**.

12. Clique no botão **Criar cluster**.

### Criar uma tarefa no Databricks

1. Na área de trabalho do Databricks, clique em “Tarefas” e “Criar tarefa”.

2. Digite um nome para a tarefa.

3. Clique em “definir jar” para abrir a caixa de diálogo “Carregar JAR para executar”.

4. Arraste o arquivo **azure-databricks-job-1.0-SNAPSHOT.jar** criado na seção **compilar o .jar para a tarefa do Databricks** para a caixa **Solte o JAR aqui para carregar**.

5. Insira **com.microsoft.pnp.TaxiCabReader** no campo **Main Class**

``` -n jar:file:/dbfs/azure-databricks-jobs/ZillowNeighborhoods-NY.zip!/ZillowNeighborhoods-NY.shp --taxi-ride-consumer-group taxi-ride-eh-cg --taxi-fare-consumer-group taxi-fare-eh-cg --window-

6. No campo argumentos, insira o seguinte:
```shell
-n jar:file:/dbfs/azure-databricks-jobs/ZillowNeighborhoods-NY.zip!/ ZillowNeighborhoods-NY.shp --taxi-ride-consumer-group taxi-ride-eh-cg --taxi-fare-consumer-group taxi-fare-eh-cg --window-interval “1 minute” --cassandra-host <nome do host Cassandra do Cosmos DB acima>
7. Instale as bibliotecas dependentes seguindo estas etapas:
    
    1. Na interface do usuário do Databricks, clique no botão **home**.
    
    2. No menu suspenso **Usuários**, clique no nome da sua conta de usuário para abrir as configurações da área de trabalho da sua conta.
    
    3. Clique na seta do menu suspenso ao lado do nome da sua conta, clique em **criar** e clique em **Biblioteca** para abrir a caixa de diálogo **Nova biblioteca**.
    
    4. No controle suspenso **Fonte**, selecione **Coordenada Maven**.
    
    5. Sob o título **Instalar artefatos Maven**, digite `com.microsoft.azure:azure-eventhubs-spark_2.11:2.3.5` na caixa de texto **Coordenada**. 
    
    6. Clique em **Criar biblioteca** para abrir a janela **Artefatos**.
    
    7. Em **Status em clusters em execução**, marque a caixa de seleção **Anexar automaticamente a todos os clusters**.
    
    8. Repita as etapas 1 a 7 para a coordenada Maven `com.microsoft.azure.cosmosdb:azure-cosmos-cassandra-spark-helper:1.0.0`.
    
    9. Repita as etapas 1 a 6 para a coordenada Maven `org.geotools:gt-shapefile:19.2`.
    
    10. Clique em **Opções avançadas**.
    
    11. Digite `http://download.osgeo.org/webdav/geotools/` na caixa de texto **Repositório**. 
    
    12. Clique em **Criar biblioteca** para abrir a janela **Artefatos**. 
    
    13. Em **Status em clusters em execução**, marque a caixa de seleção **Anexar automaticamente a todos os clusters**.

8. Adicione as bibliotecas dependentes adicionadas na etapa 7 ao trabalho criado no final da etapa 6:
 
   1. No espaço de trabalho do Azure Databricks, clique em **Jobs**.

2. Clique no nome do trabalho criado na etapa 2 da seção **criar um trabalho do Databricks**

3. Ao lado da seção **Bibliotecas dependentes**, clique em **Adicionar** para abrir a caixa de diálogo **Adicionar biblioteca dependente**. 
    
    4. Em **Biblioteca de**, selecione **Área de trabalho**.
    
    5. Clique em **usuários**, depois em seu nome de usuário e, em seguida, clique em `azure-eventhubs-spark_2.11:2.3.5`. 
    
    6. Clique em **OK**.
    
    7. Repita as etapas 1 a 6 para `spark-cassandra-connector_2.11:2.3.1` e `gt-shapefile:19.2`.

9. Ao lado de **Cluster:**, clique em **Editar**. Isso abre a caixa de diálogo **Configurar cluster**. No menu suspenso **Cluster Type**, selecione **Existing Cluster**. No menu suspenso **Select Cluster**, selecione o cluster criado na seção **criar um cluster Databricks**. Clique em **confirm**.

10. Clique em **run now**.
Execute o gerador de dados

1. Navegue até o diretório chamado `onprem` no repositório GitHub.

2. Atualize os valores no arquivo **main.env** da seguinte maneira:

```shell
    RIDE_EVENT_HUB=[Cadeia de conexão para o hub de eventos de corrida de táxi]
    FARE_EVENT_HUB=[Cadeia de conexão para o hub de eventos de tarifa de táxi]
    RIDE_DATA_FILE_PATH=/DataFile/FOIL2013
    MINUTES_TO_LEAD=0
    PUSH_RIDE_DATA_FIRST=false
    ```
    A cadeia de conexão para o hub de eventos de corrida de táxi é o valor **taxi-ride-eh** da seção de saída **eventHubs** na etapa 4 da seção *implantar os recursos do Azure*. A cadeia de conexão para o hub de eventos de tarifa de táxi é o valor **taxi-fare-eh** da seção de saída **eventHubs** na etapa 4 da seção *implantar os recursos do Azure*.

3. Execute o comando a seguir para criar a imagem do Docker.

```bash
    docker build --no-cache -t dataloader .
    ```

4. Navegue de volta para o diretório pai.

```bash
    cd ..
    ```

5. Execute o comando a seguir para executar a imagem do Docker.

    ```bash
    docker run -v `pwd`/DataFile:/DataFile --env-file=onprem/main.env dataloader:latest
    ```

A saída deve ser semelhante à seguinte:

```
Criados 10.000 registros para TaxiFare
Criados 10.000 registros para TaxiRide
Criados 20.000 registros para TaxiFare
Criados 20.000 registros para TaxiRide
Criados 30.000 registros para TaxiFare...

```

Para verificar se a tarefa do Databricks está sendo executada corretamente, abra o portal do Azure e navegue até o banco de dados Cosmos DB. Abra a guia **Data Explorer** e examine os dados na tabela **taxi records**.  

[1] <span id="note1">Donovan, Brian; Work, Dan (2016): New York City Taxi Trip Data (2010-2013). Universidade de Illinois em Urbana-Champaign. https://doi.org/10.13012/J8PN93H8
