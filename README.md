# Cloudant Kafka Connector

[![Maven Central](https://img.shields.io/maven-central/v/com.cloudant/kafka-connect-cloudant.svg)](http://search.maven.org/#search|ga|1|g:"com.cloudant"%20AND%20a:"kafka-connect-cloudant")

Este projeto inclui conectores de origem e coletor do Apache Kafka Connect para IBM Cloudant.

## Pré-lançamento

**Observação**: este arquivo README é para uma versão de pré-lançamento do
conector. Isso significa que se refere a opções de configuração e recursos que são diferentes da versão lançada atualmente. Para obter informações sobre a versão atualmente lançada, consulte o [README aqui](https://github.com/IBM/cloudant-kafka-connector/blob/0.100.2-kafka-1.0.0/README.md).

## Índice

* Configuração
* Uso

## Configuração

O conector Cloudant Kafka foi projetado para ser usado com a API Kafka Connect 3.x.

O conector Cloudant Kafka pode ser configurado em modo autônomo ou distribuído de acordo com a [documentação do conector Kafka](https://kafka.apache.org/documentation.html#connect_configuring). At a minimum it is necessary to configure:

1. `bootstrap.servers`
2. Se estiver usando um trabalhador autônomo `offset.storage.file.filename`.

### Configuração do conversor

Os padrões de distribuição kafka são geralmente os seguintes:
```
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true
```

#### Configuração do conversor: conector de origem

Para o conector de origem:
* As chaves são produzidas como `java.util.Map<String, String>` contendo uma entrada `_id` com o ID original do documento Cloudant.
* Os valores são produzidos como um (sem esquema) `java.util.Map<String, Object>`.
* Esses tipos são compatíveis com o padrão `org.apache.kafka.connect.json.JsonConverter` e devem ser compatíveis com qualquer outro conversor que aceite um `Map`.
* O `schemas.enabled` pode ser usado com segurança com um `key.converter` se desejado.
* O conector de origem não gera esquemas para os valores de registro por padrão. Para usar `schemas.enable` com `value.converter`, considere usar um registro de esquema ou o `MapToStruct` SMT detalhado abaixo.

#### Configuração do conversor: conector da pia

Para o conector da pia:
1. As chaves Kafka são atualmente ignoradas; portanto, as configurações do conversor de chave não são relevantes.
1. Assumimos que os valores em kafka são objetos JSON serializados e, portanto, `JsonConverter` é suportado.Se seus valores contiverem um esquema (`{"schema": {...}, "payload": {...}}`), defina `value.converter.schemas.enable=true`, caso contrário, defina `value .converter.schemas.enable=false`. Qualquer outro conversor que converta os valores da mensagem em tipos `org.apache.kafka.connect.data.Struct` ou `java.util.Map` também deve funcionar. No entanto, deve-se observar que a serialização subsequente dos valores `Map` ou `Struct` para documentos JSON no coletor pode não corresponder às expectativas se um esquema não tiver sido fornecido.
1. A inserção de apenas uma única revisão de qualquer `_id` é atualmente suportada.  Isso significa que não pode atualizar ou excluir documentos.
1. O campo `_rev` nos valores do evento são preservados.  Para remover `rev` durante o fluxo de dados, use `ReplaceField` Single Message Transforms (SMT).

Exemplo de configuração:
    ```
    transforms=ReplaceField
    transforms.ReplaceField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.ReplaceField.exclude=_rev
    ```
    Consulte a documentação [transformações do Kafka Connect](https://kafka.apache.org/31/documentation.html#connect_transforms) para obter mais detalhes.

**Observação:** o ID de cada documento gravado no Cloudant pelo conector do coletor pode ser configurado da seguinte forma:

1. Do valor do cabeçalho `cloudant_doc_id` no evento.  O valor passado para este cabeçalho deve ser uma string e a configuração `header.converter=org.apache.kafka.connect.storage.StringConverter` é necessária.  Isso substituirá o campo `_id` se já existir.
2. O valor do campo `_id` no JSON
3. Se nenhum outro valor não nulo ou não vazio estiver disponível, o documento será criado com um novo UUID.

####Transformações de mensagem única

As Transformações de Mensagem Única, ou SMTs, podem ser usadas para customizar campos ou valores de eventos durante o fluxo de dados. 

##### Sink
Os exemplos abaixo demonstram a modificação de campos para eventos que fluem do tópico Kafka para um banco de dados Cloudant usando o conector de coletor.

1. Se o valor do evento contiver um campo existente, não chamado `_id`, que seja adequado para uso como o ID do documento Cloudant, será possível usar a transformação `RenameField`.
    ```
    transforms=RenameField
    transforms.RenameField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.RenameField.renames=name:_id
    ```
1. Se você tiver campos `_id` e preferir que o Cloudant gere um UUID para o ID do documento, use a transformação `ReplaceField` para excluir o campo `_id` existente:
    ```
    transforms=ReplaceField
    transforms.ReplaceField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.ReplaceField.exclude=_id
    ```
1. Se você tiver eventos onde não há valor ([_tombstone_ events](https://kafka.apache.org/documentation.html#compaction)), você pode querer filtrá-los.
     - No conector do coletor Cloudant, estes podem ser indesejáveis, pois gerarão um documento vazio.
     - No conector de origem do Cloudant, os eventos de exclusão são gerados para documentos excluídos (além do próprio documento excluído).
     - Em ambos os casos, você pode usar o predicado `RecordIsTombstone` com um filtro para remover esses eventos de exclusão, conforme mostrado neste exemplo:

    ```
    transforms=dropNullRecords
    transforms.dropNullRecords.type=org.apache.kafka.connect.transforms.Filter
    transforms.dropNullRecords.predicate=isNullRecord

    predicates=isNullRecord
    predicates.isNullRecord.type=org.apache.kafka.connect.transforms.predicates.RecordIsTombstone
    ```

1. Se você quiser usar a chave do evento ou outro valor personalizado como ID do documento, use o cabeçalho personalizado `cloudant_doc_id`.
    O valor definido neste cabeçalho personalizado será adicionado ao campo `_id`. Se o campo `_id` já existir, ele será sobrescrito
    com o valor neste cabeçalho.
    Você pode usar o SMT `HeaderFrom` para mover ou copiar uma chave para o cabeçalho personalizado. O exemplo de configuração abaixo adiciona a transformação para mover
    a chave de evento `docid` para o cabeçalho personalizado `cloudant_doc_id` e define o conversor de cabeçalho como string:
   ```
   transforms=moveFieldsToHeaders
   transforms.moveFieldsToHeaders.type=org.apache.kafka.connect.transforms.HeaderFrom$Key
   transforms.moveFieldsToHeaders.fields=docid
   transforms.moveFieldsToHeaders.headers=cloudant_doc_id
   transforms.moveFieldsToHeaders.operation=move
   
   header.converter=org.apache.kafka.connect.storage.StringConverter
   ```

   **Nota**: O `header.converter` deve ser definido como `StringConverter`, pois o campo de ID do documento suporta apenas strings.

1. Se você tiver eventos onde o campo `_id` está ausente ou `null` então o Cloudant irá gerar
um ID de documento. Se você não quer que isso aconteça, defina um `_id` (veja exemplos anteriores).
Se você precisar filtrar esses documentos ou descartar campos `_id` quando o valor for `null`, você precisará criar um SMT personalizado.

**Observação**: Para qualquer um dos SMTs acima, se o campo não existir, ele deixará o evento inalterado e continuará processando o próximo evento.

##### Fonte
Os exemplos abaixo demonstram a modificação de registros produzidos pelo conector de origem Cloudant.

1. Achate mapas no documento JSON usando o `org.apache.kafka.connect.transforms.ReplaceField$Value` integrado do Kafka
    ```
    transforms=FlattenMaps
    transforms.FlattenMaps.type=org.apache.kafka.connect.transforms.Flatten
    ```

1. Achate matrizes no documento JSON usando `com.ibm.cloud.cloudant.kafka.connect.transforms.ArrayFlatten`. Observe que esta transformação
    é adequado apenas para uso com valores de registro de Mapa e filtrará registros que não estejam em conformidade. Como tal, se usado em conjunto com o
    Transformação `MapToStruct`, esta operação `ArrayFlatten` deve preceder `MapToStruct` no pipeline SMT.
    A propriedade de configuração `delimiter` pode ser usada para personalizar o delimitador, cujo padrão é `.`.
     ```
    transforms=FlattenArrays
    transforms.FlattenArrays.type=com.ibm.cloud.cloudant.kafka.connect.transforms.ArrayFlatten
    ```

1. Converta valores `java.util.Map` sem esquema em `org.apache.kafka.connect.data.Struct` com um esquema inferido. Esta transformação é projetada
    para melhorar a compatibilidade com outros conectores e conversores que requerem um registro do tipo `Struct`. Para esquemas complexos, um registro de esquema
    deve ser usado.
    ```
    transforms=MapToStruct
    transforms.MapToStruct.type=com.ibm.cloud.cloudant.kafka.connect.transforms.MapToStruct
    ```

1. Omita documentos de design dos eventos produzidos usando o `org.apache.kafka.connect.transforms.Filter` integrado do Kafka
    em conjunto com o predicado `com.ibm.cloud.cloudant.kafka.connect.transforms.predicates.IsDesignDocument`. Observe que isso
    O predicado depende do formato chave dos eventos do conector de origem do Cloudant, portanto deve ser aplicado antes de qualquer outra transformação que
    altere o formato da chave.
    ```
    transforms=omitDesignDocs
    transforms.omitDesignDocs.type=org.apache.kafka.connect.transforms.Filter
    transforms.omitDesignDocs.predicate=isDesignDoc

    predicates=isDesignDoc
    predicates.isDesignDoc.type=com.ibm.cloud.cloudant.kafka.connect.transforms.predicates.IsDesignDocument
    ```

### autenticação

Para ler ou gravar no Cloudant, algumas propriedades de autenticação precisam ser configuradas. Essas propriedades são comuns ao conector de origem e ao coletor.

A number of different authentication methods are supported. IAM authentication is the default and recommended method; see [locating your service credentials](https://cloud.ibm.com/docs/Cloudant?topic=Cloudant-locating-your-service-credentials) for details on how to find your IAM API key.

#### cloudant.auth.type

O método (ou tipo) de autenticação. Este valor não diferencia maiúsculas de minúsculas.

O valor padrão é `iam`.

Os valores válidos são:

- `iam`
- `couchdb_session`
- `basic`
- `noAuth`
- `bearerToken`
- `container`
- `vpc`.

Com exceção de `noAuth`, cada um desses métodos de autenticação requer a configuração de uma ou mais propriedades adicionais. Eles estão listados abaixo.
#### cloudant.apikey

Para uso com autenticação `iam`.

#### cloudant.username, cloudant.password

Para uso com autenticação `couchdb_session` ou `basic`.

#### cloudant.bearer.token

Para uso com autenticação `bearerToken`.

#### cloudant.iam.profile.id 

Para uso com autenticação `container` ou `vpc`.

#### cloudant.iam.profile.name

Para uso com autenticação `container`.

#### cloudant.cr.token.filename

Para uso com autenticação `container`.

#### cloudant.iam.profile.crn

Para uso com autenticação `vpc`.

#### cloudant.auth.url, cloudant.scope, cloudant.client.id, cloudant.client.secret

Para uso com autenticação `iam`, `container` ou `vpc`.

### Cloudant as source

Além das propriedades relacionadas à autenticação, o conector de origem Cloudant suporta as seguintes propriedades:

Parâmetro | Valor                                                                  | Requerido | Valor Default | Descrição
---:|:-----------------------------------------------------------------------|:---|:---|:---
nome| cloudant-source                                                        |YES|None|Um nome exclusivo para identificar o conector.
connector.class| com.ibm.cloud.cloudant.kafka.SourceChangesConnector |YES|None|The connector class name.
topics| \<topic1\>,\<topic2\>,..                                               |YES|None|A list of topics you want messages to be written to.
cloudant.url| https://\<uuid\>.cloudantnosqldb.appdomain.cloud                       |YES|None|The Cloudant server to read documents from.
cloudant.db| \<your-db\>                                                            |YES|None|The Cloudant database to read documents from.
cloudant.since| 1-g1AAAAETeJzLYWBgYMlgTmGQT0lKzi9..                                    |NO|0|The first change sequence to process from the Cloudant database above. 0 will apply all available document changes.
batch.size| 400                                                                    |NO|1000|The batch size used to bulk read from the Cloudant database.
cloudant.omit.design.docs| false                                                                  |NO|false| Set to true to omit design documents from the messages produced.
cloudant.value.schema.struct| false                                                                  |NO|false| _EXPERIMENTAL_ Set to true to generate a `org.apache.kafka.connect.data.Schema.Type.STRUCT` schema and send the Cloudant document payload as a `org.apache.kafka.connect.data.Struct` using the schema instead of the default of a string of the JSON document content when using the Cloudant source connector.
cloudant.value.schema.struct.flatten| false                                                                  |NO|false| _EXPERIMENTAL_ Set como true para nivelar matrizes e objetos aninhados do documento Cloudant durante a geração de estrutura. Usado apenas quando cloudant.value.schema.struct é verdadeiro e permite o processamento de matrizes JSON com tipos de elementos mistos ao usar essa opção.

#### Exemplo

Para ler um banco de dados Cloudant como origem e gravar documentos em um tópico Kafka, aqui está um `connect-cloudant-source.properties` mínimo, usando a autenticação IAM padrão:
```
name=cloudant-source
connector.class=com.ibm.cloud.cloudant.kafka.SourceChangesConnector
topics=mytopic
cloudant.url=https://some-uuid.cloudantnosqldb.appdomain.cloud
cloudant.db=my-db
cloudant.apikey=my-apikey
```

### Cloudant as sink

Além das propriedades relacionadas à autenticação, o conector de coletor Cloudant suporta as seguintes propriedades:

Parâmetro | Valor | Required | Default value | Descrição
---:|:---|:---|:---|:---
name|cloudant-sink|YES|None|A unique name to identify the connector with.
connector.class|com.ibm.cloud.cloudant.kafka.SinkConnector|YES|None|The connector class name.
topics|\<topic1\>,\<topic2\>,..|YES|None|The list of topics you want to consume messages from.
cloudant.url|https://\<your-account\>.cloudant.com|YES|None|The Cloudant server to write documents to.
cloudant.db|\<your-db\>|YES|None|The Cloudant database to write documents to.
tasks.max|5|NO|1|The number of concurrent threads to use for parallel bulk insert into Cloudant.
batch.size|400|NO|1000|The maximum number of documents to commit with a single bulk insert.
replication|false|NO|false|Managed object schema in sink database <br>*true: duplicate objects from source <br>false: adjust objects from source (\_id = [\<topic-name\>\_\<partition\>\_\<offset>\_\<sourceCloudantObjectId\>], kc\_schema = Kafka value schema)*

#### Exemplo

Para consumir mensagens de um tópico Kafka e salvar como documentos em um banco de dados Cloudant, aqui está um `connect-cloudant-sink.properties` mínimo, usando a autenticação IAM padrão:

```
name=cloudant-sink
connector.class=com.ibm.cloud.cloudant.kafka.SinkConnector
topics=mytopic
cloudant.url=https://some-uuid.cloudantnosqldb.appdomain.cloud
cloudant.db=my-db
cloudant.apikey=my-apikey
```

## Usage

From version `0.200.0` the cloudant-kafka-connector jar is available to download from the [releases page](https://github.com/IBM/cloudant-kafka-connector/releases).

O arquivo jar contém o plugin e as dependências não-Kafka necessárias para execução. Uma vez copiado em um
[configurado `plugin.path`](https://kafka.apache.org/documentation.html#connectconfigs_plugin.path) de uma instalação do Kafka 3.x ele estará disponível para uso.

A execução do conector no Kafka está disponível por meio de scripts no caminho de instalação do Kafka:

`$kafka_home/bin/connect-standalone.sh` or `$kafka_home/bin/connect-distributed.sh`

Use os arquivos de configuração apropriados para execução autônoma ou distribuída com o Cloudant como origem, coletor ou ambos.

Por exemplo:
- execução autônoma com Cloudant como fonte:

  ```
  $kafka_home/bin/connect-standalone.sh connect-standalone.properties connect-cloudant-source.properties
  ```

- standalone execution with Cloudant as sink:

  ```
  $kafka_home/bin/connect-standalone.sh connect-standalone.properties connect-cloudant-sink.properties
  ```

- standalone execution with multiple configurations, one using Cloudant as source and one using Cloudant as sink:

  ```
  $kafka_home/bin/connect-standalone.sh connect-standalone.properties connect-cloudant-source.properties connect-cloudant-sink.properties
  ```

Any number of connector configurations can be passed to the executing script.

INFO level logging is configured by default to the console. To change log levels or settings, work with

`$kafka_home/config/connect-log4j.properties`

e adicione configurações de log como

`log4j.logger.com.ibm.cloud.cloudant.kafka=DEBUG, stdout`
