
## Apache Pinot Cluster

### Docker
docker-compose per inizializzazione cluster Apache Pinot e Apache Kafka su sistemi WINDOWS, LINUX e MACOS con processori NON Apple Silicon (M1, M2, ..)

```bash
  docker-compose --project-name pinot up
```

Per avviare il cluster Apache Pinot su MacOS con processori Apple Silicon (M1, M2, ..) utilizzare il seguente comando:

```bash
  docker-compose -f docker-compose-AppleSilicon.yml --project-name pinot up
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
      "name": "datetime",
      "dataType": "LONG",
      "format": "1:MILLISECONDS:EPOCH",
      "granularity": "1:MILLISECONDS"
    }
  ]
}
```

Una volta creato lo schema si crei la tabella Realtime tramite il pulsante "Add Realtime Table" secondo le seguenti impostazioni:

```bash
{
  "REALTIME": {
    "tableName": "powerConsumption_REALTIME",
    "tableType": "REALTIME",
    "segmentsConfig": {
      "schemaName": "powerConsumption",
      "replication": "1",
      "retentionTimeUnit": "DAYS",
      "retentionTimeValue": "2",
      "replicasPerPartition": "1",
      "timeColumnName": "datetime",
      "minimizeDataMovement": false
    },
    "tenants": {
      "broker": "DefaultTenant",
      "server": "DefaultTenant",
      "tagOverrideConfig": {}
    },
    "tableIndexConfig": {
      "invertedIndexColumns": [],
      "noDictionaryColumns": [],
      "rangeIndexColumns": [],
      "rangeIndexVersion": 2,
      "autoGeneratedInvertedIndex": false,
      "createInvertedIndexDuringSegmentGeneration": false,
      "sortedColumn": [],
      "bloomFilterColumns": [],
      "loadMode": "MMAP",
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
      "onHeapDictionaryColumns": [],
      "varLengthDictionaryColumns": [],
      "enableDefaultStarTree": false,
      "enableDynamicStarTreeCreation": false,
      "aggregateMetrics": false,
      "nullHandlingEnabled": false,
      "optimizeDictionary": false,
      "optimizeDictionaryForMetrics": false,
      "noDictionarySizeRatioThreshold": 0.85
    },
    "metadata": {},
    "quota": {},
    "routing": {},
    "query": {},
    "ingestionConfig": {
      "segmentTimeValueCheck": true,
      "continueOnError": false,
      "rowTimeValueCheck": false
    },
    "isDimTable": false
  }
}
```

Assicurarsi di inserire correttamente il valore di "stream.kafka.broker.list" su "pinot-kafka:9093" e "stream.kafka.topic.name" su "power".

Così facendo si crea automaticamente un topic Kafka di nome "power" su cui verranno inviati i dati dallo script python e ricevuti dal cluster di Pinot.

### Installazione libreria kafka-python

``` bash
pip3 install kafka-python
``` 

### Installazione libreria pinotdb

``` bash
pip3 install pinotdb
``` 

Per inviare i dati alla tabella di Pinot si può utilizzare lo script python (kafka_random_threading.py) che genera valori casuali e li invia ogni secondo con il timestamp attuale alla tabella di Pinot tramite il topic Kafka appena generato.
Oltre a generare i dati casuali aggiorna ogni volta il file che consente la visualizzazione dell'ultimo valore inserito nel database sulla pagina web e rimane in attesa di query da parte della pagina web per inoltrarle al database Pinot e ricevere la risposta da visualizzare sulla pagina.

Per eseguire questo file aprire una finestra del terminale con i privilegi di amministratore ed eseguire il comando:

``` bash
python3 kafka_random_threading.py
```

kafka_random_threading.py
``` bash
from pinotdb import connect
import time
import json
import os
from threading import Thread
import datetime
import random
import json
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='localhost:9092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))

def rand_last():
    while True:
        ts = round(time.time()) + 7200
        
        val = random.randint(0,1000)
        print("datatetime:"+str(ts)+" - sensor:random - powerValue:"+str(val))
        producer.send('power', {"datetime":ts , "sensor" : "random" , "powerValue" : val})

        outfile = open("files/last.json","w")
        strfile = [ ts , val , 'piSensor']
        json.dump(strfile, outfile)
        outfile.close()

        time.sleep(2)

conn = connect(host='localhost', port=8099, path='/query/sql', scheme='http')
curs = conn.cursor()

def query():
    while True:
        if(os.path.exists("files/query.txt")):
            f = open("files/query.txt", "r")
            query = f.read()
            print(query)
            f.close()
            outfile = open("files/response.json","w")
            try:
                curs.execute(query)
                for row in curs:
                    print(row)
                    json.dump(str(row), outfile, indent=2)
                    outfile.write('\n')
                print()
            except:
                json.dump("ERROR", outfile, indent=2)
            outfile.close()
            os.remove("files/query.txt")

# create two new threads
t1 = Thread(target=rand_last)
t2 = Thread(target=query)

# start the threads
t1.start()
t2.start()

# wait for the threads to complete
t1.join()
t2.join()
```

In alternativa si può utilizzare il file kafka_mqtt_threading.py per inserire nel database Pinot i dati reali provenienti da un sensore di corrente ricevuti tramite MQTT.

## Apache Superset (solo WINDOWS, LINUX e MACOS NON APPLE SILICON)

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
Aggiungere anche le seguenti righe nel file superset/docker/pythonpath_dev/superset_config.py:
``` bash
PUBLIC_ROLE_LIKE = 'Gamma'

TALISMAN_ENABLED = False
ENABLE_CORS = True
HTTP_HEADERS={"X-Frame-Options":"ALLOWALL"} 
```
Questo serve a fornire i permessi necessari per la visualizzazione del grafico sulla pagina HTML.

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

Esempio di Line Chart di dati casuali:
![image](https://github.com/OnestiFilippo/ApachePinot/assets/77025139/4e366137-4cc8-4764-9c2d-34fc3018772b)

## Pagina Web Apache Pinot

Con l'esecuzione del docker-compose iniziale si avvia anche il server web Apache sulla porta 8080 per la visualizzazione della pagina web.
Per accedervi collegarsi all'indirizzo:

http://localhost:8080

Da questa è possibile effettuare query al database Pinot e ricevere la risposta sulla stessa pagina, visualizzare l'ultimo valore inserito nel database che si aggiorna in tempo reale e visualizzare il grafico di tutti i valori presenti nel database generato tramite Apache Superset.

Screenshot pagina web:
<img width="1680" alt="f3d79d12-7b11-4b59-82e3-c1786fc752c6" src="https://github.com/OnestiFilippo/Progetto_ApachePinot/assets/77025139/5f715d30-fe15-4976-a6b7-fd31574e7211">

