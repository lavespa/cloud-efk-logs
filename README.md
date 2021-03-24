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


Eseguiamo il file "kube-logging.yaml" per creare il nuovo namespace:


```yml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging

```

Creiamo il name space con:

              kubectl create -f kube-logging.yaml

Eseguendo il controllo
               
			  kubectl get namespaces
			  
dovrebbe essere presente li nuovo namespace "kube-logging".


## Step2 - Creazione di Elasticsearch come StatefulSet
Ora che abbiamo creato uno spazio dei nomi per ospitare il nostro stack di registrazione, possiamo iniziare a distribuire i suoi vari componenti. 
Inizieremo innanzitutto distribuendo un cluster Elasticsearch con 1 nodo.

### Creazione di un Service tipo Headless
Per iniziare, creeremo un servizio Kubernetes headless chiamato elasticsearch che definirà un dominio DNS per il nostro pod di elasticsearch. Un servizio headless non 
esegue il bilanciamento del carico o ha un IP statico.

Eseguiamo il file "elasticsearch_svc.yaml" per creare il servizio per accedere al pod elasticsearch:

```yml
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

Definiamo un Service chiamat "elasticsearch" nel "kube-logging" Namespace e le assegniamo all'app il nome elasticsearch come etichetta label. 
Quindi impostiamo nelle specifiche come selector sull'applicazione app come elasticsearch in modo che il servizio selezioni i pod con l'app 
elasticsearch label. Quando associamo il nostro Elasticsearch StatefulSet a questo servizio, il servizio restituirà il record DNS(links, in questo caso 1) 
che puntano a Elasticsearch Pod con l' app: elasticsearch label.



Quindi impostiamo clusterIP: None, il che rende il servizio di tipo headless. Infine, definiamo le porte 9200 e 9300 che vengono utilizzate 
per interagire con l'API REST e per la comunicazione tra i nodi.

Creiamo il servizio:
       
	          kubectl create -f elasticsearch_svc.yaml

Infine, ricontrolla che il servizio sia stato creato con successo utilizzando:

              kubectl get svc --namespace=kube-logging

Ora che abbiamo configurato il nostro servizio headless e il dominio elasticsearch.kube-logging.svc.cluster.local  per il nostro pod, 
possiamo andare avanti e creare lo StatefulSet per ElasticSearch.

### Creazione dello StatefulSet

Un Kubernetes StatefulSet consente di assegnare un'identità stabile ai pod e di garantire loro uno spazio di archiviazione stabile e persistente. 
Elasticsearch richiede una memoria stabile per rendere persistenti i dati durante il riavvio del pod. 
I Deployment resources in K8s sono pensati per l'utilizzo senza stato e sono piuttosto leggeri. Gli StatefulSet vengono utilizzati quando lo stato 
deve essere mantenuto. Pertanto, questi ultimi utilizzano dei volumeClaimTemplates/ cioè volumi persistenti per garantire che possano mantenere lo 
stato durante i riavvii dei componenti.

Eseguiamo il file "elasticsearch_statefulset.yaml" così definito: 

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: kube-logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.2.0
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pvc-persistent-cfg
```

Effettuiamo il deploy dello StatefulSet :

              kubectl create -f elasticsearch_statefulset.yaml

È possibile monitorare StatefulSet mentre viene distribuito utilizzando kubectl rollout status:

              kubectl rollout status sts/es-cluster --namespace=kube-logging

Una volta che tutti i pod sono stati installati e quindi in Running, puoi verificare che il tuo cluster Elasticsearch funzioni correttamente 
eseguendo una richiesta di tipo REST.

Per fare ciò, ènecessario effettuare una port-forward dalla porta locale 9200 alla porta 9200 su uno dei nodi Elasticsearch ( es-cluster-0) 
utilizzando kubectl port-forward:

              kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging

A questo aprendo una shell di Git Bash, eseguiamo:

              curl http://localhost:9200/_cluster/state?pretty

e se otteniamo una response 200(OK) allora è tutto ok. Nella respinse ottenuta sostanzialmente possiamo verificare che il nostro gruppo
k8s-logs è stato creato correttamente con il nodo es-cluster-0.
La configurazione di Elasticsearch è quindi attiva e funzionante.  

