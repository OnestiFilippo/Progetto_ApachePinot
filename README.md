
## Apache Pinot Cluster

### Docker
docker-compose per inizializzazione cluster Apache Pinot e Apache Kafka

```bash
  docker-compose --project-name pinot up
```
Una volta avviato il cluster collegarsi all'indirizzo:

http://localhost:9000

Per creare lo schema e la tabella per l'inserimento dei dati in Pinot recarsi nella sezione Tables e creare lo schema tramite il pulsante "Add Schema":

```bash
{
  "schemaName": "powerConsumption",
  "dimensionFieldSpecs": [
    {
      "name": "sensor",
      "dataType": "STRING"
    }
  ],
  "metricFieldSpecs": [
    {
      "name": "powerValue",
      "dataType": "INT"
    }
  ],
  "dateTimeFieldSpecs": [
    {
      "name": "timestamp",
      "dataType": "STRING",
      "format": "1:MILLISECONDS:EPOCH",
      "granularity": "1:MILLISECONDS"
    }
  ]
}
```

Una volta creato lo schema si crei la tabella Realtime tramite il pulsante "Add Realtime Table":

```bash
{
  "REALTIME": {
    "tableName": "powerConsumption_REALTIME",
    "tableType": "REALTIME",
    "segmentsConfig": {
      "schemaName": "powerConsumption",
      "replication": "1",
      "replicasPerPartition": "1",
      "minimizeDataMovement": false,
      "timeColumnName": "timestamp"
    },
    "tenants": {
      "broker": "DefaultTenant",
      "server": "DefaultTenant",
      "tagOverrideConfig": {}
    },
    "tableIndexConfig": {
      "streamConfigs": {
        "streamType": "kafka",
        "stream.kafka.topic.name": "power",
        "stream.kafka.broker.list": "pinot-kafka:9093",
        "stream.kafka.consumer.type": "lowlevel",
        "stream.kafka.consumer.prop.auto.offset.reset": "smallest",
        "stream.kafka.consumer.factory.class.name": "org.apache.pinot.plugin.stream.kafka20.KafkaConsumerFactory",
        "stream.kafka.decoder.class.name": "org.apache.pinot.plugin.stream.kafka.KafkaJSONMessageDecoder",
        "realtime.segment.flush.threshold.rows": "0",
        "realtime.segment.flush.threshold.time": "24h",
        "realtime.segment.flush.threshold.segment.size": "100M"
      },
      "invertedIndexColumns": [],
      "noDictionaryColumns": [],
      "aggregateMetrics": false,
      "nullHandlingEnabled": false,
      "optimizeDictionary": false,
      "optimizeDictionaryForMetrics": false,
      "noDictionarySizeRatioThreshold": 0.85,
      "rangeIndexColumns": [],
      "rangeIndexVersion": 2,
      "autoGeneratedInvertedIndex": false,
      "createInvertedIndexDuringSegmentGeneration": false,
      "sortedColumn": [],
      "bloomFilterColumns": [],
      "loadMode": "MMAP",
      "onHeapDictionaryColumns": [],
      "varLengthDictionaryColumns": [],
      "enableDefaultStarTree": false,
      "enableDynamicStarTreeCreation": false
    },
    "metadata": {},
    "quota": {},
    "routing": {},
    "query": {},
    "ingestionConfig": {
      "continueOnError": false,
      "rowTimeValueCheck": false,
      "segmentTimeValueCheck": true
    },
    "isDimTable": false
  }
}
```

Assicurarsi di inserire correttamente il valore di "stream.kafka.broker.list" su "pinot-kafka:9093" e "stream.kafka.topic.name" su "power".

Così facendo si crea un topic Kafka di nome topic su cui verranno inviati i dati dallo script python e ricevuti dal cluster di Pinot.

Per inviare i dati alla tabella di Pinot si può utilizzare uno script python che genera valori casuali e li invia alla tabella di Pinot tramite il topic Kafka appena generato.

``` bash
from kafka import KafkaProducer
import json
import random
import time
import datetime

producer = KafkaProducer(bootstrap_servers='localhost:9092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))

while True:
    ts = datetime.datetime.now().strftime("%d/%m/%Y - %H:%M:%S")
    producer.send('power', {"timestamp":ts , "sensor" : "piSensor" , "powerValue" : random.randint(0,1000)})
    time.sleep(1)
```

## Apache Superset

Per l'inizializzazione di Apache Superset eseguire il seguente comando per la clonazione della repository di Superset:
``` bash
git clone https://github.com/apache/superset.git
```
spostarsi nella cartella appena creata:
``` bash
cd superset
```
e creare il file "requirements-local.txt" per inizializzare la libreria per la connessione tra Apache Pinot e Apache Superset:
``` bash
touch ./docker/requirements-local.txt
echo "pinotdb" >> ./docker/requirements-local.txt
```
Lanciare il comando per la ricostruzione del docker-compose:
``` bash
docker-compose build --force-rm
```
Avviare Apache Superset tramite docker-compose 
``` bash
docker-compose -f docker-compose-non-dev.yml pull
docker-compose -f docker-compose-non-dev.yml up
```

Una volta avviato Superset aprire la pagina http://localhost:8088 e accedere con le credenziali di default:
``` bash
username: admin
password: admin
```
A questo punto connettere il database di Apache Pinot a Apache Superset tramite i seguenti passaggi:
Dalla homepage di Superset:
Settings -> Database Connections -> +DATABASE
Dal menu a tendina selezionare Apache Pinot
Nel campo SQLALCHEMY URI inserire:
``` bash
pinot://XXX.XXX.XXX.XXX:8099/query/sql?controller=http://XXX.XXX.XXX.XXX:9000
```
Sostituendo al posto di XXX.XXX.XXX.XXX l'indirizzo IP locale della macchina in cui è in esecuzione il cluster Pinot.

Ora è possibile creare i propri Charts e Dashboard selezionando la tabella desiderata.

![image](https://github.com/OnestiFilippo/ApachePinot/assets/77025139/871b3305-c8e4-4d82-ba8e-4a4952d7aa3b)

