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

Traduzido com a versão gratuita do tradutor - DeepL.com
