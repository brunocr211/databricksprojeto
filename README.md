# Processamento de fluxo com Azure Databricks

Esta arquitetura de referência mostra um pipeline de [processamento de fluxo](/azure/architecture/data-guide/big-data/real-time-processing) e ponta a ponta. Esse tipo de pipeline possui quatro estágios: ingestão, processamento, armazenamento e análise e geração de relatórios.
Para esta arquitetura de referência, o pipeline ingere dados de duas fontes, realiza uma junção entre registros relacionados de cada fluxo, enriquece o resultado e calcula uma média em tempo real. Os resultados são armazenados para análises posteriores.

![](https://github.com/mspnp/architecture-center/blob/master/docs/reference-architectures/data/images/stream-processing-databricks.png)

**Cenário:** uma empresa de táxi coleta dados sobre cada viagem. Para este cenário, assumimos que existem dois dispositivos distintos enviando dados. O táxi possui um taxímetro que envia informações sobre cada corrida — duração, distância e locais de embarque e desembarque. Um dispositivo separado aceita pagamentos dos clientes e envia dados sobre as tarifas.
Para identificar tendências de uso, a empresa de táxi deseja calcular a gorjeta média por milha percorrida, em tempo real, para cada bairro.
