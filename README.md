# cloud-efk-logs
How to configure E elasticsearch, F Fluentd and K Kibana (EFK) for logging on K8s clusters


## EFK
Quando abbiamo in esecuzione più servizi e applicazioni all'interno di un cluster K8s, diventa sempre
più forte e pressante l'esigenza di avere uno stack applicativo di logging centralizzato a livello 
di cluster che sia in grado di ordinare ed analizzare in modo rapido un volume anche grosso di data-logging
prodotti nei nostri pod. Una soluzione molto utilizzata in K8s a questo tipo di problema è proprio l'utilizzo 
dello stack EFK.


## Elasticsearch

Elasticsearch è un motore di ricerca in tempo reale, distribuito e scalabile che consente la ricerca e l'analisi
di testo. Viene comunemente utilizzato per indicizzare e cercare in grandi volumi di data-logging, ma può anche 
essere utilizzato per cercare diversi tipi di documenti.


## Kibana

Elasticsearch viene comunemente distribuito insieme a Kibana, un potente frontend e dashboard per la visualizzazione dei dati 
per Elasticsearch. Kibana consente di esplorare i dati di log di Elasticsearch tramite un'interfaccia web, creare dashboard 
e query per estrarre rapidamente i data-logging interessati e ottenere informazioni dettagliate sulle tue applicazioni Kubernetes.

## Fluentd

Utilizzeremo infine Fluentd per raccogliere, trasformare e inviare i data logging al backend di Elasticsearch. Fluentd
è un infatti un popolare data-collector open source facilmente configurabile sui nostri nodi Kubernetesmil cui scopo 
è quello di intercettare, analizzare, filtrare e infine ttasformare i file di logging dei vari containr presenti sui
container. Fondamentalmente quindi ogni container Fluentd non fa altro che leggere la cartella /vat/lib/docker per
ottenere i log di ogni container sul nodo di J8s ed inviarli quindi a ElasticSearch. Infine, quando accediamo a Kibana
, richiediamo i log salvati sul backend di ElasticSearch. Possiamo riassumere con la seguente immagine, senza
perderci in altre parole:


<img width="880" alt="Diagram TraceId con Sleuth" src="https://github.com/lavespa/cloud-efk-logs/blob/main/image/EFK_Stack.png">

## Step1 - Creazione namespace

Prima di implementare un cluster Elasticsearch, creeremo prima uno spazio dei nomi in cui installeremo tutte le risorse k8s 
coinvolte nella configurazione dello stack per il logging.. Kubernetes ci consente di separare gli oggetti in esecuzione 
nel tuo cluster utilizzando un'astrazione "virtual cluster" chiamata Namespaces. Creiamo quindi uno "kube-logging" namespace 
in cui installeremo i componenti dello stack EFK. Questo namespace tra l'altro ci consentirà inoltre di pulire e rimuovere 
rapidamente lo stack di logging senza alcuna perdita di funzionalità per il cluster Kubernetes.


Eseguiamo il file kube-logging.yaml per creare il nuovo namespace:


```yml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging

```
