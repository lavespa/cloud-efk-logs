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
ottenere i log di ogni container sul nodo di K8s ed inviarli quindi a ElasticSearch. Infine, quando accediamo a Kibana
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

Eseguendo il controllo:
               
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

Definiamo un Service chiamato "elasticsearch" nel "kube-logging" Namespace e le assegniamo all'app il nome elasticsearch come etichetta label. 
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

Poichè quindi è necessario che la configurazione di Elasticsearch sia persistente, cioè cooservi il suo stato al riavvio
del pod affinchè possa essere applicata la stessa configurazione iniziale, definiamo un Persistent Volume Clain tramite il
provider di archiviazione di default di k8s per creare un Persistence Volume da montare(mount) sul pod in questo caso di 
Elkasticsearch.

Eseguiamo il file "pvc.yaml" così definito:

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-persistent-cfg
  namespace: kube-logging
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
  #storageClassName: yourClass
```

              kubectl create -f pvc.yaml
			 
e controllo

              kubectl get svc --namespace=kube-logging

Specifichiamo quindi la sua modalità di accesso come ReadWriteOnce, il che significa che può essere montato solo come 
lettura-scrittura da un singolo nodo.
Infine specifichiamo che vorremmo che ogni PersistentVolume avesse una dimensione di 50Mb (È necessario regolare questo valore in base alle 
proprie esigenze di produzione.)
Se non si specifica il storageClassName, il PVC utilizzerà, come detto in precedenza, la classe di archiviazione predefinita 
per creare il PV e collegarlo.

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

Esaminiamo per blocchi la configurazione di Elasticsearch come Statefulset:


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
```

In questo blocco definiamo uno StatefulSet chiamato "es-cluster" in "kube-logging" namespace. Quindi lo associamo al nostro 
Servizio "elasticsearch" creato in precedenza utilizzando il campo serviceName. Ciò garantisce che ogni pod nello StatefulSet 
sarà accessibile utilizzando il seguente indirizzo DNS:, es-cluster-[0].elasticsearch.kube-logging.svc.cluster.local dove [0]
corrisponde al numero ordinale intero assegnato al pod. Questa configurazione si presta ad essere replicata su più pod.

Specifichiamo 1 replicas(Pods) e impostiamo il matchLabels selettore su app: elasticseach, che poi specifichiamo nella spec.template.metadata sezione. 
I campi spec.selector.matchLabels e spec.template.metadata.labels devono corrispondere.

Possiamo ora passare alle specifiche dell'oggetto:

```yml
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
```

Qui definiamo i pod nello StatefulSet. 

Assegniamo un nome all'unico container ,"elasticsearch", e scegliamo docker.elastic.co/elasticsearch/elasticsearch:7.2.0 come immagine 
Docker per ElasticSearch.
Utilizziamo quindi il "resources" campo per specificare che il container necessita di almeno 0,1 vCPU garantite e può esplodere 
fino a 1 vCPU (il che limita l'utilizzo delle risorse del pod quando si esegue un carico iniziale di grandi dimensioni o si ha a che fare 
con un picco di carico. Per settings e info leggere la [Documentazione](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) 
ufficiale di Kubernetes.

Quindi apriamo e specifichiamo le porte 9200 e 9300 rispettivamente per l'API REST e la comunicazione tra nodi. Specifichiamo un volumeMount chiamato "data" che monterà il 
PersistentVolume denominato "data" nel contenitore nel percorso /usr/share/elasticsearch/data. 
Abbiamo infatti definito in pvc.yaml un VolumeClaims per questo StatefulSet che abbiamo specificatp più avanti di questo blocco sotto volumes.

Infine, impostiamo alcune variabili d'ambiente nel contenitore:

- cluster.name: Il nome del cluster Elasticsearch, che in questo caso ho settato a k8s-logs;
- node.name: Il nome del nodo che abbiamo impostato sul campo metadata.name utilizzando valueFrom. Questo è in pratica equivalente a es-cluster-[0];
- discovery.seed_hosts: Questo campo ho deciso di aggiungerlo se si decide di utilizzare più repliche sul nodo. Esso infatti imposta 
un elenco di nodi idonei per il master nel cluster che eseguiranno il seeding del processo di rilevamento dei nodi. Grazie al servizio headless che abbiamo 
configurato in precedenza, nel caso in cui si definisse un set di 3 repliche ad esempio, i nostri Pod avranno i seguenti domini: 
es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local. Utilizzando la risoluzione DNS di Kubernetes del namingspace locale, 
possiamo abbreviarla in es-cluster-[0,1,2].elasticsearch. Per settings e info leggere la [Documentazione](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/discovery-settings.html) 
ufficiale di Kubernetes.
- cluster.initial_master_nodes: Questo campo specifica un elenco di nodi idonei che parteciperanno al processo di elezione del nodo master.
- ES_JAVA_OPTS: Impostato su -Xms512m -Xmx512m che dice alla JVM di utilizzare una dimensione di heap minima e massima di 512 MB. È necessario regolare questi 
parametri in base alla disponibilità e alle esigenze delle risorse del cluster.

Il prossimo blocco da analizzare è il seguente:

```yml
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
```

In questo blocco definiamo diversi container di inizializzazione che vengono eseguiti prima del container principale, cioè "elasticsearch" . Questi container di inizializzazione 
vengono eseguiti con dei command fino al completamento nell'ordine in cui sono definiti.

Il primo, denominato "fix-permissions", esegue un chown(Linux) comando per modificare il proprietario e il gruppo della directory dei dati di Elasticsearch in 1000:1000, l'UID dell'utente di Elasticsearch.
Questo perchè per impostazione predefinita, Kubernetes monta la directory dei dati come root, il che la rende inaccessibile a Elasticsearch.

Il secondo, denominato "increase-vm-max-map", esegue un comando per aumentare i limiti del sistema operativo sui conteggi mmap, che per impostazione predefinita 
potrebbero essere troppo bassi, con conseguenti errori di memoria insufficiente. Per questa configurazione vedere la [Documentazione](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html) 
su Elastic.

Il prossimo InitContainer da eseguire è "increase-fd-ulimit", che esegue il comando ulimit per aumentare il numero massimo di file aperti. In sostanza gli
ulimits sono uno strumento di ottimizzazione delle prestazioni delle applicazioni linux, il cui scopo è quello di limitare l'utilizzo delle risorse di un programma.
Ad esempio nel caso di un cluster di microservizi (pods+service db+etc....) è possibile che vengano aperte migliaia di file mappati in memoria per gestire ad esempio
migliaia di connessioni di rete. L'uso di ulimits è quindi essenziale per ottenere prestazioni adeguate nelle risorse K8s
Tutte le info su qesta parte relativa alla scalabilità di Elasticsearch sono reperibili nella seguente [Documentazione](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_notes_for_production_use_and_defaults).

Infine viene abilitata nel pod per tutti i container di inizializzazione la modalità "privileged" attraverso il securityContext, che ci permette 
di poter essere visti come amministratori nella modifica dei vari elementi di gestione del pod(manipolazione dello stack di rete, accesso a cartelle....)


L'ultimo blocco è quello relativo alla parte di storage da associare a ciascuna delle risorse coinvolte nel pod di Elasticsearch;
questo storage comne già detto in precedenza è quello già creato precedentemente con il PersistenceVolumeClaims del file pvc.yaml:

```yml
volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pvc-persistent-cfg
```

Nel nodo "volumes" specifichiamo come detto come PVC il claim pvc-persistent-cfg definito in pvc.yaml il cui scopo è quello di creare
dei volumi dati persistenti per il/i pod definiti qui.


Effettuiamo il deploy dello StatefulSet :

              kubectl create -f elasticsearch_statefulset.yaml

È possibile monitorare StatefulSet mentre viene distribuito utilizzando kubectl rollout status:

              kubectl rollout status sts/es-cluster --namespace=kube-logging

Una volta che tutti i pod sono stati installati e quindi in Running, puoi verificare che il tuo cluster Elasticsearch funzioni correttamente 
eseguendo una richiesta di tipo REST.

Per fare ciò, ènecessario effettuare una port-forward dalla porta locale 9200 alla porta 9200 su uno dei nodi Elasticsearch ( es-cluster-0) 
utilizzando kubectl port-forward:

              kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging

A questo punto aprendo una shell di Git Bash, eseguiamo:

              curl http://localhost:9200/_cluster/state?pretty

e se otteniamo una response 200(OK) allora è tutto ok. Nella response ottenuta sostanzialmente possiamo verificare che il nostro gruppo
k8s-logs è stato creato correttamente con il nodo es-cluster-0.
La configurazione di Elasticsearch è quindi attiva e funzionante.  

## Step3 - Creazione di Kibana Deployment e Service

Per avviare Kibana su Kubernetes, creeremo un servizio chiamato kibana e una resource Deployment composta da una replica del pod. 
E' possibile ridimensionare il numero di repliche in base alle proprie esigenze di produzione e, facoltativamente, specificare 
un LoadBalancer per il servizio allo scopo di bilanciare il carico delle richieste tra i pod di distribuzione.

Questa volta creeremo il servizio e la distribuzione nello stesso file.

```yml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  selector:
    app: kibana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.2.0
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
```

In questa specifica abbiamo definito un servizio chiamato "kibana" nel kube-logging namespace e gli abbiamo dato l'app: kibana come label.
Abbiamo anche specificato che dovrebbe essere accessibile sulla porta 5601 e utilizzare l'app: kibana come label per essere selezionata 
dai pod di destinazione del servizio.

Nelle specifiche del Deployment , definiamo una distribuzione chiamata kibana e specifichiamo che vorremmo 1 replica Pod.
Specifichiamo le stesse risorse definite in Elasticsearch nel nodo "resources".

Usiamo l'image docker.elastic.co/kibana/kibana:7.2.0(equivalente versione di elastic per evirae disallineamenti).

Successivamente utilizziamo la variabile di ambiente ELASTICSEARCH_URL  per impostare l'endpoint e la porta per il cluster Elasticsearch. 
Utilizzando Kubernetes DNS, questo endpoint corrisponde al nome del servizio elasticsearch.

Infine, impostiamo la porta del contenitore di Kibana su 5601; su questa il kibana Service inoltrerà le richieste esterne.

Possiamo quindi eseguire il servizio e il deploy di Kibana:

              kubectl create -f kibana.yaml
			  
Verifichiamo con:

              kubectl rollout status deployment/kibana --namespace=kube-logging

Se il pod di Kibana è in Running a questo punto possiamo accedere all'interfaccia Kibana effettuando un port-forward da una porta locale al nodo Kubernetes che esegue Kibana. 
Copiamo e incolliamo il name del Pod di Kibana in running  tramite:

              kubectl get pods --namespace=kube-logging

ed eseguiamo un port-forward dalla porta locale 5601 alla porta 5601 su questo pod:

              kubectl port-forward kibana-yyyyyyyyyy-xxxxx 5601:5601 --namespace=kube-logging
			  
A questo punto dovremmo visualizzare la dashboard di benvenuto di Kibana correttamente nel nostro browser:

              [http://localhost:5601](http://localhost:5601)
			  
## Step4 - Creazione di Fluentd come DaemonSet

Sebbene Kubernetes non fornisca una soluzione nativa per il logging a livello di cluster, esistono diversi approcci comuni che possiamo prendere in considerazione.
Quello che noi seguiremo è l'utilizzo di un logging-agent a livello di nodo che viene eseguito su ogni nodo. Il logging-agent è in sostanza uno
strumento dedicato(Fluentd) che espone i logs o effettua il push dei logs ad un backend(Elasticsearch). Quindi il logging-agent è solitamente un container 
che ha accesso ad una directory con i file di logs di tutti i container delle applicazioni su quel nodo.
Affinchè questo logging-agent sia eseguito su ogni nodo, Kubernetes consiglia di eseguirlo come DaemonSet. Il logging quindi a livello di nodo è possibile 
dunque creando un singolo agent-logging su ogni nodo e sopratutto non richiede nessuna modifica su tutte le applicazioni in esecuzione in quel nodo.
Tutti i container delle applicazioni quindi scriveranno i loro logs(staout e stderr), senza un formato concordato, e il logging-agent a livello di nodo
li raccoglierà e l'inoltrerà in modo aggregato verso un backend.
Il nostro logging-agent sarà in questo caso proprio Fluentd.

Eseguiamo il file "fluentd.yaml" così definito: 

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.kube-logging.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Esaminiamo per blocchi la configurazione del logging-aget Fluentd.

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
```

Qui creiamo un ServiceAccount chiamato fluentd che i/il Pod Fluentd utilizzeranno per accedere all'API Kubernetes. Lo creiamo nel kube-logging namespace e ancora una volta gli diamo come label app: fluentd.

Creiamo adesso il nodo ClusterRole:

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
```

Definiamo la risorsa ClusterRole chiamata fluentd a cui concediamo i permessii/autorizzazioni get, list e watch per gli oggetti pod e namespaces . 
ClusterRoles in sostanza ti consente di concedere l'accesso alle risorse Kubernetes con ambito cluster come ad esempio i nodi.
Per la [Documentazione](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).


A questo punto creiamo e definiamo un CustomRoleBinding:

```yml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-logging
```

La risorsa ClusterRoleBinding chiamata "fluentd" lega il ruolo fluentd del cluster all'account del servizio fluentd. 
Questo permetterà di concedere al fluentd ServiceAccount le autorizzazioni elencate nel ClusterRole fluentd.

Infine possiamo analizzare la parte rimanente di Fluentd come DaemonSet, cioè il vero e proprio logging-agent in funzione su ogni nodo del cluster.

```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
```
	
In questo blocco definiamo un DaemonSet chiamato fluentd nel kube-logging namespace e gli diamo come label app: fluentd.

Successivamente definiamo le sue specifiche:

```yml
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.kube-logging.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
```

Abbiniamo in questo caso la label app: fluentd definita in metadata.labels e assegniamo al DaemonSet il ServiceAccount "fluentd". 
Selezioniamo e quindi individuiamo questo Pod anche con  app: fluentd, gestito in questo caso come un DaemonSet.


Successivamente definiamo una NoSchedule tolleranza abbinata alla taint(contaminazione) equivalente sui nodi master Kubernetes. In sostanza di
default non sarebbe possibile eseguire il DaemonSet sul nodo master del nostro cluster, questo garantisce invece che sia possibile 
distribuire il DaemonSet di Fluentd anche sul nodo master del cluster. Questo a mio parere aumenta la scalabilità, qualora sia necessario 
è possibile eliminare questa tolleranza facendo in modo che il Pod Fluentd non sia eseguibile sui nodi master.

Usiamo poi l'image Debian v1.4.2 ufficiale fornita dai manutentori di Fluentd. Il Dockerfile e il contenuto di questa immagine sono disponibili nel 
seguente repository:

              [https://github.com/fluent/fluentd-kubernetes-daemonset/tree/master/docker-image/v1.4/debian-elasticsearch](https://github.com/fluent/fluentd-kubernetes-daemonset/tree/master/docker-image/v1.4/debian-elasticsearch)
			  
Infine configuriamo Fluentd utilizzando alcune variabili di ambiente al fine di permettere la scrittura dei log sul backend di Elasticsearch:

- FLUENT_ELASTICSEARCH_HOST: Abbiamo impostato l'indirizzo del Service di tipo headless "elasticsearch" definito in precedenza: elasticsearch.kube-logging.svc.cluster.local;
- FLUENT_ELASTICSEARCH_PORT: Abbiamo impostato il valore della porta elasticsearch che abbiamo configurato in precedenza, 9200;
- FLUENT_ELASTICSEARCH_SCHEME: Lo abbiamo impostato su http;
- FLUENTD_SYSTEMD_CONF: Lo impostiamo su disable per sopprimere l'output correlato al systemd. In sostanza se non si imposta systemd(log system) nel container, fluentd mostra 
tutti i messaggi di funzionamento del sistema.

Infine abbiamo l'ultimo blocco:

```yml
resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

In quest'ultimo blocco specifichiamo un limite di memoria di 512 MiB sul Pod di FluentD e garantiamo 0,1 vCPU(nel calcolo della memoria virtuale 1G=1VCPU) e 200 MiB di memoria per request. È possibile regolare 
questi limiti e richieste di risorse in base al volume di logging previsto e alle risorse disponibili.

Dopo effettuiamo il mount dei volumi nei quali Fluentd andrà a pescare i log per effettuare l'aggregazione e precisamente nei path /var/log e /var/lib/docker/containers dei containers usando 
le variabili varlog e varlibdockercontainers nel nodo volumeMounts. Definite le directory(mount) su cui persistere i logs i loro volumi di persistenza vengono configurati alla fine del blocco.

L'ultimo parametro che definiamo in questo blocco è "terminationGracePeriodSeconds", che dà a Fluentd 30 secondi per spegnersi in modo naturale dopo aver ricevuto un SIGTERM segnale. 
Dopo 30 secondi, ai contenitori viene inviato un segnale SIGKILL. Il valore predefinito per terminationGracePeriodSeconds è 30s, quindi nella maggior parte dei casi questo parametro può essere anche omesso.

Eseguiamo aesso il DaemonSet con :

              kubectl create -f fluentd.yaml
			  
Verifichiamo che il nostro DaemonSet sia stato implementato correttamente:

kubectl get ds --namespace=kube-logging

Se il nostro logging-agent è in esecuzione sull'unico pod (1) del nostro cluster che corrisponde al nostro unico nodo nel cluster, possiamo a questo punto controllare e verificare
con Kibana che i dati dei logs vengano raccolti e inviati correttamente ad Elasticsearch..

Per verificare questo andiamo su 

              ["http://localhost:5601"](http://localhost:5601)

e clicchiamo su Discover nel menu di navigazione laterale sx.
Definiamo quindi un "index pattern" per Elasticsearch, utilizzando ad esempio il pattern "logstash-*" che ci permette di acquisire tutti i data log nel nostro cluster di Elasticsearch.


Inseriamo quindi nella casella di testo "logstash-*" e clicchiamo su "Next step". Alla pagina successiva inseriamo un filtro che verrà utilizzato da Kibana per filtrare tutti i data log
presenti in base ad esempio al timestamp. Nel menù a discesa quindi inserirò @timestamp e cliccherò su "Create index pattern".


Clicchiamo nuovamente su Discover e dovremmo a questo punto visualizzare le stringhe di log recenti prodotte dal nostro Pod.

A questo punto abbiamo configurato e implementato correttamente lo stack EFK sul nostro cluster Kubernetes.
Per l'utilizzo di Kibana è disponibile la seguente [Documentazione](https://www.elastic.co/guide/en/kibana/current/index.html)

 

