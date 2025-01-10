# Keycloak with Podman (e Kind)
### Pochi semplici comandi per eseguire 2 (o N) istanze di Keycloak ed un database PostgresQL.

## TL;DR

clonate il repo GIT e fate partire l'ambiente usando *podman compose*:

```shell
$ git clone https://github.com/aleoncini/keycloak-with-podman.git
Cloning into 'keycloak-with-podman'...
remote: Enumerating objects: 86, done.
remote: Counting objects: 100% (86/86), done.
remote: Compressing objects: 100% (53/53), done.
remote: Total 86 (delta 34), reused 74 (delta 25), pack-reused 0 (from 0)
Receiving objects: 100% (86/86), 24.09 KiB | 6.02 MiB/s, done.
Resolving deltas: 100% (34/34), done.
$ cd keycloak-with-podman
$ docker compose up -d
97feb7c6973da609ca1e4c45a6eaa890aee9cbd2504f23053131a3a5f2fc5460
dbf8949840c5dab4d92dcb5423998b8d7cc01f769ee97bfe9a1cec3fbf17d870
489fdd64aae645b87087c2bba7de067017cbe5626d6c22fbddf4dbde03e3d606
d2dffbc137b86314fd4f23206f0e1dd1f7609dc18f135206d680ea534640f175
b0aad119a74f6266e774c91e9b796ad090409bfaa73fc3092a3dcd6b2a73b4ae
2616d63a1feb0a54c6c72299e102e2a058e6959589cf58ae1574da57e8c6020f
$
```

Accedete con il browser alla console di Keycloak [localhost:8080](http://localhost:8080)

Volendo si puo' anche accedere all'amministrazione di Postgres [localhost:8088](http://localhost:8088)

Per fermare l'ambiente e rimuovere tutti i container creati

```shell
podman compose down
```
> ℹ️ **_INFO_**  
> Se dovessi incontrare degli errori con Podman potrebbe essere necessario abilitare l'accesso al socket da systemd:  
> 
```shell
sudo systemctl enable --now podman.service
```
## Deploy di un cluster Keycloak 

Può essere utile in un ambiente di sviluppo istanziare un server Keycloak per per gestire il controllo degli accessi a servizi applicativi. Ci sono vari modi per farlo, installandolo sulla propria workstation o utilizzando un container e sicuramente tool come Podman o Docker ci tornano utili e di fatto con un singolo comando abbiamo la possibilità di avere un server Keycloak pronto all'uso.

L'istanza singola tuttavia non permette di eseguire certi tipi di test, per esempio di alta affidabilità, o analizzare comportamenti con configurazioni diciamo "simil produzione". Anche in questo caso Podman ci può aiutare permettendo l'installazione di un ambiente più sofisticato e più simile ad una configurazione di produzione. In questo laboratorio anziche' il singolo nodo di Keycloak istanzieremo un cluster di 2 (o piu') nodi in alta affidabiltà con persistenza su un database Postgres. Proveremo anche diverse modalità di esecuzione, a partire dalla creazione dei singoli container, passando per Podman Compose (la modalità sopra descritta), per arrivare ad una modalità *"Kubernetes"* tramite **Kind**.

Project structure:
```
.
├── kc
│   └── Dockerfile
├── lb
│   └── nginx.conf
├── kube-deployment
│   ├── ingress.yaml
│   ├── kc-deployment.yaml
│   ├── pg-deployment.yaml
│   ├── pg-storage.yaml
│   ├── pgadmin-deployment.yaml
│   ├── secrets.yaml
│   ├── tls.crt
│   └── tls.key
├── compose.yaml
└── README.md
```

## manually run each single container

Prima di tutto ci serve creare una rete interna, i due nodi di Keycloak infatti condividono le sessioni web attraverso un cluster Infinispan che si forma automaticamente mediante un sistema di auto-discovery che cerca tutti i nodi in esecuzione sulla stessa rete. Inoltre entrambi i nodi devono dialogare con il DB Postgres che contiene le configurazioni di Keycloak.

cominciamo quindi con il comando:

```shell
podman network create kc-network
```

Naturalmente potete chiamare la rete come meglio credete.
Provate ad eseguire il comando

```shell
podman network ls
```

dovreste avre un output del tipo:

```shell
NETWORK ID    NAME          DRIVER
2f259bab93aa  podman        bridge
6a5df8616cf3  kc-network    bridge
```

Ora scarichiamo le immagini che ci servono:

```shell
podman pull docker.io/library/postgres:latest
```

```shell
podman pull quay.io/keycloak/keycloak:latest
```

Per primo faremo partire il Database Postgres ma prima di lanciare il comando per eseguirlo creiamo anche un volume per la persistenza dei dati che andranno sul database:

```shell
podman volume create pgdata
```

verificate che il volume sia stato correttamente creato e verificate anche il mount point:

```shell
podman volume inspect pgdata
```

a questo punto possiamo eseguire il database, usate:

```shell
podman run -d --name pgsql --net kc-network -p 5432:5432 -v pgdata:/var/lib/postresql/data -e POSTGRES_USER=dbuser -e POSTGRES_PASSWORD=change_me -e POSTGRES_DB=keycloak postgres
```

> ℹ️ **_INFO_**  
> Potreste anche tralasciare di creare un volume e quindi anche eliminare l'opzione **-V** nel comando precedente, ricordatevi però che in quel caso se fermate il pod che contiene il DB e poi lo eseguite di nuovo perderete le configurazioni salvate nella sessione precedente (che potrebbe anche essere un caso d'uso voluto).  
> 

Sempre al fine di controllare che tutto stia funzionando correttamente potreste usare il comando: 

```shell
podman exec pgsql env
```

che vi ritornerà una lista di variabili tra cui dovreste vedere anche quelle inserite nel comando precedente.

Altro comando che vi tornerà utile è

```shell
podman network inspect kc-network
```

in cui potrete ricavare l'indirizzo IP assegnato da Podman al container che esegue PostgresQL (e che dovrete usare qualora vi colleghiate alla console di amministrazione PGADMIN).

## \[Opzionale\] Download and Start pgAdmin

Questa parte non è necessaria, la riporto perché in generale può far comodo, in ambiente di sviluppo, ispezionare il database delle configurazioni di Keycloak. Se volete provare aggiungete questi comandi, altrimenti saltate a [`Build and Run Keycloak`](#build).

```shell
podman pull docker.io/dpage/pgadmin4:latest
```

```shell
podman run -d --rm --name pgadmin -p 8082:80 --net kc-network -e PGADMIN_DEFAULT_EMAIL="dbuser@example.com" -e PGADMIN_DEFAULT_PASSWORD="change_me" pgadmin4
```

Potete adesso usare la web console di Postgres usando il browser ed aprendo la pagina [localhost:8082](http://localhost:8082)

> ℹ️ **_INFO_**  
> Per usare la console di amministrazione del database inserite dbuser@example.com/change_me come credenziali. Nella dashboard cliccate su new server.
> nella form inserite un nome per la sessione poi andate al tab connection. In questo tab inserite l'indirizzo di rete che avete letto per il container pgsql come output di inspect della rete.
> utilizzate dbuser/change_me come credenziali e cliccate su save. Dovreste a questo punto ritrovarvi nella dashboard del database postgres avviato precedentemente.
>

## Build and Run Keycloak

Con riferimento a questa [guida](https://www.keycloak.org/server/containers) la prima cosa che dobbiamo fare è costruire una immagine locale personalizzata di Keycloak a partire da quella ufficiale scaricata all'inizio.

Questo passaggio è molto importante perché ci permette di aggiungere al Keycloak base tutti i plugin, provider ed ogni qualunque altro tipo di personalizzazione di cui necessita il nostro server Keycloak, che poi un Keycloak personalizzato è il caso più frequente.

In questa fase potete aggiungere ogni tipo di personalizzazione prevista per il keycloak del vostro ambiente (es. provider SPID).

Per questo esempio io ho allegato un *Dockerfile* con cui specifico alcuni dei parametri di connessione al DB e preparo dei certificati self-signed per le connessioni HTTPS.

> ***NOTA:*** La strategia qui può essere di vario tipo, per esempio specificando le variabili da riga di comando. Il compose proposto aggiunge queste informazioni
> tramite variabili di ambiente. In questo esempio, avendo deciso di buildare l'immagine di keycloak
> di riferimento potrebbe essere consigliabile aggiungerle in fase di build. Se decidete di fare in questo modo **dovete togliere i commenti nel Dockerfile** ed inserire l'indirizzo IP del DB che potete ricavare dal comando *podman network inspect kc-network*.

```shell
cd kc
```

```shell
podman build . -t kc
```

Ora tutto è pronto per lanciare le due (o più) istanze di Keycloak:

```shell
podman run -d --rm --net kc-network -p 8443:8443 -p 9000:9000 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=change_me kc start --optimized --hostname=localhost
```

Per la seconda istanza (o successive) non dovete far altro che ripetere il comando precedente facendo attenzione a cambiare le porte di riferimento

```shell
podman run -d --rm --net kc-network -p 8543:8443 -p 9100:9000 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=change_me kc start --optimized --hostname=localhost
```

Potete adesso accedere alla web console di keycloak usando il browser ed aprendo la pagina su [localhost:8443](https://localhost:8443)
oppure [localhost:8543](https://localhost:8543)

## \[Opzionale\] Aggiungiamo un bilanciatore nell'architettura

Le ultime versioni di Keycloak sono state progettate ed implementate con lo specifico scopo di essere eseguite in un ambiente Cloud Oriented o Cloud Compliant, in particolare la progettazione del runtime, basato su Quarkus, trova la sua collocazione ideale sulla piattaforma Red Hat Openshift. Su Openshift è possibile utilizzare Keycloak in modalità **auto-scaling** che permette di aggiungere automaticamente dei pod in base al carico di lavoro che insiste sulla componente Keycloak. In questo contesto l'accesso alle istanze (quindi il bilanciamento delle richieste sui nodi che compongono il cluster Keycloak) è regolato da appositi meccanismi di routing messi a disposizione da Openshift. Se nel nostro ambientino di sviluppo locale volessimo aggiungere anche una componente che simula questa funzionalità (per esempio per testare il meccanismo di bilanciamento e le politiche appropriate) occorre aggiungere un ulteriore pod che si occupi di bilanciare le richieste al nostro mini-cluster Keycloak. Per questo esempio useremo un bilanciatore software sfruttando le funzionalità offerte da NGINX.

Prima di tutto fermiamo i nostri server Keycloak (usate i nomi reali che leggete dal primo comando):

```shell
podman ps
podman stop <kc-1> <kc-2>
```

e rilanciamo i server con lo stesso comando senza però il parametro **-p** che espone le porte dei nostri pod (in questo caso rimuovo anche il nome del pod ed aggiungo l'utilizzo della porta 8080).
Normalmente la porta 8080 è disabilitata, tuttavia nel momento in cui non esponiamo direttamente il server possiamo configurarlo in HTTP.

```shell
podman run -d --rm --net kc-network -e KC_HTTP_ENABLED=true -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=change_me kc start --optimized --hostname=http://localhost:8080
```

Ripetete di nuovo lo stesso comando (in questo caso è proprio lo stesso comando in quanto non serve nemmeno cambiare le porte esposte) per avere due istanze di Keycloak (se volete lanciatene anche di più). Controllate che i pod siano up & running 

```shell
podman ps
```

I nodi keycloak lanciati in questo modo non sono raggiungibili direttamente usando il browser locale. Controllate quindi gli indirizzi di rete interna che sono stati asseganti:

```shell
podman network inspect kc-network
```

prendete nota del parametro **ipnet** dei container in quanto lo dovrete inserire nella configurazione del file *nginx.conf* allegata. Dopo aver sistemato la configurazione buildate la vostra immagine nginx.

```shell
cd lb
```
```shell
podman build -t kc-nginx .
```

ed eseguite:

```shell
podman run -d --rm --net kc-network -p 8080:80 kc-nginx
```

Se adesso accedete dal browser alla pagina http://localhost:8080 sarete in realtà connessi con uno dei due server Keycloak. Potete anche fare delle cose e poi provare a stoppare uno dei container di keycloak, vedrete che potrete continuare a lavorare sull'altro. Buon lavoro!!

## Deploy locale utilizzando Kind

> ℹ️ **_NOTA:_**  
> I file qui utilizzati come esempio per un laboratorio locale potrebbero essere utilizzati anche su ambienti evoluti come
> [Red Hat Openshift](https://www.redhat.com/en/technologies/cloud-computing/openshift), per questo tipo di ambienti però è consigliabile
> l'uso dell'[operator](https://operatorhub.io/operator/keycloak-operator). La procedura qui descritta è un riferimento utile solo per ambienti locali di sviluppo.

Ci sono diversi ambienti di tipo "Local Kubernetes" come per esempio minikube, k3s e k3d. Kind ha la caratteristica di essere piuttosto leggero e, personalmente, è quello che preferisco ed è quello che ho utilizzato per questo laboratorio.

Per prima cosa aggiungo un namespace in modo da raggruppare le varie componenti necessarie per il deploy:

```shell
kubectl create namespace sso
```

Adesso potremo aggiungere le componenti che servono al nostro deploy utilizzando alcuni file YAML

Per prima cosa aggiungo alcuni secrets che saranno poi utilizzati dai vari server (Keycloak e PostgresQL).

```shell
kubectl apply -f secrets.yaml -n sso
```

non fate caso al warning che vedrete in merito alla mancanza dei dati di tipo PEM, infatti se controllate

```shell
kubectl get secrets -n sso
```

vedrete che sono stati creati 3 secrets differenti.

Per quanto riguarda postgres, in analogia con quanto già fatto creando direttamente i container tramite podman, dobbiamo prima definire un'area storage per dare persistenza al database:

```shell
kubectl apply -f pg-storage.yaml -n sso
```

il file definisce sia un PeristentVolume che il relativo PersistentVolumeClaim.

A questo punto possiamo deployare postgres

```shell
kubectl apply -f pg-deployment.yaml -n sso
```

**[Opzionale]** Come per l'esempio del Composer potreste voler utilizzare la console di amministrazione del database, troverete un file YAML che crea un pod con l'applicazione *PGADMIN* ed il relativo servizio. Se volete provare a collegarvi dovete a questo punto attivare il portforwarding del servizio creato

```shell
kubectl get svc -n sso
```

verificate che ci sia una riga del tipo:

```shell
pgadmin    ClusterIP      10.96.106.117   <none>        8088/TCP         51m
```

ed eseguite il comando

```shell
kubectl -n sso port-forward service/pgadmin 8082:8080
```

Per accedere aprite una pagina del vostro browser su [localhost:8082](http://localhost:8082) ed utilizzate le credenziali: **dbuser@example.com/change_me**

Se tutto ha funzionato vi troverete all'interno della console web di Postgres, aggiungete un server di riferimento che potrete chiamare keycloak (parametri di riferimento: host=postgres, user=dbuser, password=change_me).

Non resta che deployare Keycloak, fermate (Ctrl-C) il portforwarding ed eseguite

```shell
kubectl apply -f kc-deployment.yaml
```

anche in questo caso per accedere alla console occorre riattivare il corrispondente portforwarding con 

```shell
kubectl -n sso port-forward service/keycloak 8080:8080
```

e sempre tramite il vostro browser accedete a [localhost:8080](http://localhost:8080) ed utilizzate le credenziali: **admin/change_me**



