# Cloudant Kafka Connector

[![Maven Central](https://img.shields.io/maven-central/v/com.cloudant/kafka-connect-cloudant.svg)](http://search.maven.org/#search|ga|1|g:"com.cloudant"%20AND%20a:"kafka-connect-cloudant")

Este projeto inclui conectores de origem e coletor do Apache Kafka Connect para IBM Cloudant.

## Pré-lançamento

**Observação**: este arquivo README é para uma versão de pré-lançamento do
conector. Isso significa que se refere a opções de configuração e recursos que são diferentes da versão lançada atualmente. Para obter informações sobre a versão atualmente lançada, consulte o [README aqui](https://github.com/IBM/cloudant-kafka-connector/blob/0.100.2-kafka-1.0.0/README.md).

## Status de lançamento

Experimental

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
1. O campo `_rev` nos valores do evento são preservados.  To remove `rev` during data flow, use the `ReplaceField` Single Message Transforms (SMT).
Example configuration:
    ```
    transforms=ReplaceField
    transforms.ReplaceField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.ReplaceField.exclude=_rev
    ```
    See the [Kafka Connect transforms](https://kafka.apache.org/31/documentation.html#connect_transforms) documentation for more details.

**Observação:** o ID de cada documento gravado no Cloudant pelo conector do coletor pode ser configurado da seguinte forma:

1. From the value of the `cloudant_doc_id` header on the event.  The value passed to this header must be a string and the `header.converter=org.apache.kafka.connect.storage.StringConverter` config is required.  This will overwrite the `_id` field if it already exists.
1. The value of the `_id` field in the JSON
1. If no other non-null or non-empty value is available the document will be created with a new UUID.

####Transformações de mensagem única

As Transformações de Mensagem Única, ou SMTs, podem ser usadas para customizar campos ou valores de eventos durante o fluxo de dados. 

##### Sink
Os exemplos abaixo demonstram a modificação de campos para eventos que fluem do tópico Kafka para um banco de dados Cloudant usando o conector de coletor.

1. If the event value contains an existing field, not called `_id`, that is suitable to use as the Cloudant document ID, then you can use the `RenameField` transform.
    ```
    transforms=RenameField
    transforms.RenameField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.RenameField.renames=name:_id
    ```
1. If you have `_id` fields and would prefer to have Cloudant generate a UUID for the document ID, use the `ReplaceField` transform to exclude the existing `_id` field:
    ```
    transforms=ReplaceField
    transforms.ReplaceField.type=org.apache.kafka.connect.transforms.ReplaceField$Value 
    transforms.ReplaceField.exclude=_id
    ```
1. If you have events where there is no value ([_tombstone_ events](https://kafka.apache.org/documentation.html#compaction)), you may wish to filter these out.
    - In the Cloudant sink connector, these may be undesirable as they will generate an empty document.
    - In the Cloudant source connector, tombstone events are generated for deleted documents (in addition to the deleted document itself).
    - In either case, you can use the `RecordIsTombstone` predicate with a filter to remove these tombstone events as shown in this example:

    ```
    transforms=dropNullRecords
    transforms.dropNullRecords.type=org.apache.kafka.connect.transforms.Filter
    transforms.dropNullRecords.predicate=isNullRecord

    predicates=isNullRecord
    predicates.isNullRecord.type=org.apache.kafka.connect.transforms.predicates.RecordIsTombstone
    ```

1. If you want to use the event key or another custom value as the document ID then use the `cloudant_doc_id` custom header.
   The value set in this custom header will be added to the `_id` field.  If the `_id` field already exists then it will be overwritten
   with the value in this header.
   You can use the `HeaderFrom` SMT to move or copy a key to the custom header. The example config below adds the transform to move 
   the `docid` event key to the `cloudant_doc_id` custom header and sets the header converter to string:
   ```
   transforms=moveFieldsToHeaders
   transforms.moveFieldsToHeaders.type=org.apache.kafka.connect.transforms.HeaderFrom$Key
   transforms.moveFieldsToHeaders.fields=docid
   transforms.moveFieldsToHeaders.headers=cloudant_doc_id
   transforms.moveFieldsToHeaders.operation=move
   
   header.converter=org.apache.kafka.connect.storage.StringConverter
   ```

   **Note**: The `header.converter` is required to be set to `StringConverter` since the document ID field only supports strings.

1. If you have events where the `_id` field is absent or `null` then Cloudant will generate
a document ID. If you don't want this to happen then set an `_id` (see earlier examples).
If you need to filter out those documents or drop `_id` fields when the value is `null` then you'll need to create a custom SMT.

**Note**: For any of the SMTs above, if the field does not exist it will leave the event unmodified and continue processing the next event.

##### Source
The examples below demonstrate modifying records produced by the Cloudant source connector.

1. Flatten maps in the JSON document using the Kafka built-in `org.apache.kafka.connect.transforms.ReplaceField$Value`
    ```
    transforms=FlattenMaps
    transforms.FlattenMaps.type=org.apache.kafka.connect.transforms.Flatten
    ```

1. Flatten arrays in the JSON document using `com.ibm.cloud.cloudant.kafka.connect.transforms.ArrayFlatten`. Note that this transform
   is only suitable for use with Map record values and will filter records that do not conform. As such if used in conjunction with the
   `MapToStruct` transform, this `ArrayFlatten` operation must precede `MapToStruct` in the SMT pipeline.
   The `delimiter` configuration property may be used to customize the delimiter, which defaults to `.`.
    ```
    transforms=FlattenArrays
    transforms.FlattenArrays.type=com.ibm.cloud.cloudant.kafka.connect.transforms.ArrayFlatten
    ```

1. Convert schemaless `java.util.Map` values to `org.apache.kafka.connect.data.Struct` with an inferred schema. This transform is designed
   to improve compatibility with other connectors and converters that requires a `Struct` type record. For complex schemas a schema registry
   should be used.
    ```
    transforms=MapToStruct
    transforms.MapToStruct.type=com.ibm.cloud.cloudant.kafka.connect.transforms.MapToStruct
    ```

1. Omit design documents from the produced events by using the Kafka built-in `org.apache.kafka.connect.transforms.Filter`
   in conjunction with the predicate `com.ibm.cloud.cloudant.kafka.connect.transforms.predicates.IsDesignDocument`. Note that this
   predicate relies on the key format of the Cloudant source connector events so must be applied before any other transformations that
   alter the key format.
    ```
    transforms=omitDesignDocs
    transforms.omitDesignDocs.type=org.apache.kafka.connect.transforms.Filter
    transforms.omitDesignDocs.predicate=isDesignDoc

    predicates=isDesignDoc
    predicates.isDesignDoc.type=com.ibm.cloud.cloudant.kafka.connect.transforms.predicates.IsDesignDocument
    ```

### Authentication

In order to read from or write to Cloudant, some authentication properties need to be configured. These properties are common to both the source and sink connector.

A number of different authentication methods are supported. IAM authentication is the default and recommended method; see [locating your service credentials](https://cloud.ibm.com/docs/Cloudant?topic=Cloudant-locating-your-service-credentials) for details on how to find your IAM API key.

#### cloudant.auth.type

The authentication method (or type). This value is case insensitive.

The default value is `iam`.

Valid values are:

- `iam`
- `couchdb_session`
- `basic`
- `noAuth`
- `bearerToken`
- `container`
- `vpc`.

With the exception of `noAuth`, each of these authentication methods requires one or more additional properties to be set. These are listed below.

#### cloudant.apikey

For use with `iam` authentication.

#### cloudant.username, cloudant.password

For use with `couchdb_session` or `basic` authentication.

#### cloudant.bearer.token

For use with `bearerToken` authentication.

#### cloudant.iam.profile.id 

For use with `container` or `vpc` authentication.

#### cloudant.iam.profile.name

For use with `container` authentication.

#### cloudant.cr.token.filename

For use with `container` authentication.

#### cloudant.iam.profile.crn

For use with `vpc` authentication.

#### cloudant.auth.url, cloudant.scope, cloudant.client.id, cloudant.client.secret

For use with `iam`, `container`, or `vpc` authentication.

### Cloudant as source

In addition to those properties related to authentication, the Cloudant source connector supports the following properties:

Parameter | Value                                                                  | Required | Default value | Description
---:|:-----------------------------------------------------------------------|:---|:---|:---
name| cloudant-source                                                        |YES|None|A unique name to identify the connector with.
connector.class| com.ibm.cloud.cloudant.kafka.SourceChangesConnector |YES|None|The connector class name.
topics| \<topic1\>,\<topic2\>,..                                               |YES|None|A list of topics you want messages to be written to.
cloudant.url| https://\<uuid\>.cloudantnosqldb.appdomain.cloud                       |YES|None|The Cloudant server to read documents from.
cloudant.db| \<your-db\>                                                            |YES|None|The Cloudant database to read documents from.
cloudant.since| 1-g1AAAAETeJzLYWBgYMlgTmGQT0lKzi9..                                    |NO|0|The first change sequence to process from the Cloudant database above. 0 will apply all available document changes.
batch.size| 400                                                                    |NO|1000|The batch size used to bulk read from the Cloudant database.
cloudant.omit.design.docs| false                                                                  |NO|false| Set to true to omit design documents from the messages produced.
cloudant.value.schema.struct| false                                                                  |NO|false| _EXPERIMENTAL_ Set to true to generate a `org.apache.kafka.connect.data.Schema.Type.STRUCT` schema and send the Cloudant document payload as a `org.apache.kafka.connect.data.Struct` using the schema instead of the default of a string of the JSON document content when using the Cloudant source connector.
cloudant.value.schema.struct.flatten| false                                                                  |NO|false| _EXPERIMENTAL_ Set to true to flatten nested arrays and objects from the Cloudant document during struct generation. Only used when cloudant.value.schema.struct is true and allows processing of JSON arrays with mixed element types when using that option.

#### Example

To read from a Cloudant database as source and write documents to a Kafka topic, here is a minimal `connect-cloudant-source.properties`, using the default IAM authentication:

```
name=cloudant-source
connector.class=com.ibm.cloud.cloudant.kafka.SourceChangesConnector
topics=mytopic
cloudant.url=https://some-uuid.cloudantnosqldb.appdomain.cloud
cloudant.db=my-db
cloudant.apikey=my-apikey
```

### Cloudant as sink

In addition to those properties related to authentication, the Cloudant sink connector supports the following properties:

Parameter | Value | Required | Default value | Description
---:|:---|:---|:---|:---
name|cloudant-sink|YES|None|A unique name to identify the connector with.
connector.class|com.ibm.cloud.cloudant.kafka.SinkConnector|YES|None|The connector class name.
topics|\<topic1\>,\<topic2\>,..|YES|None|The list of topics you want to consume messages from.
cloudant.url|https://\<your-account\>.cloudant.com|YES|None|The Cloudant server to write documents to.
cloudant.db|\<your-db\>|YES|None|The Cloudant database to write documents to.
tasks.max|5|NO|1|The number of concurrent threads to use for parallel bulk insert into Cloudant.
batch.size|400|NO|1000|The maximum number of documents to commit with a single bulk insert.
replication|false|NO|false|Managed object schema in sink database <br>*true: duplicate objects from source <br>false: adjust objects from source (\_id = [\<topic-name\>\_\<partition\>\_\<offset>\_\<sourceCloudantObjectId\>], kc\_schema = Kafka value schema)*

#### Example

To consume messages from a Kafka topic and save as documents into a Cloudant database, here is a minimal `connect-cloudant-sink.properties`, using the default IAM authentication:

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

The jar file contains the plugin and the non-Kafka dependencies needed to run. Once copied into a
[configured `plugin.path`](https://kafka.apache.org/documentation.html#connectconfigs_plugin.path) of a Kafka 3.x installation it will be available for use.

Connector execution in Kafka is available through scripts in the Kafka install path:

`$kafka_home/bin/connect-standalone.sh` or `$kafka_home/bin/connect-distributed.sh`

Use the appropriate configuration files for standalone or distributed execution with Cloudant as source, as sink, or both.

For example:
- standalone execution with Cloudant as source:

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

and add log settings like

`log4j.logger.com.ibm.cloud.cloudant.kafka=DEBUG, stdout`
