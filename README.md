# Keycloak with Podman
Pochi semplici comandi per eseguire 2 (o N) istanze di Keycloak ed un database PostgresQL.

In un ambiente di sviluppo è molto facile istanziare un server Keycloak, lo si può scaricare ed eseguire in locale sulla propria workstation, o, ancora più semplicemente, se si ha a disposizione podman, si può eseguire una immagine ufficiale di Keycloak. Quindi di fatto serve un solo comando per avere Keycloak up & running sul proprio PC.

L'istanza singola tuttavia non permette di eseguire certi tipi di test, per esempio di alta affidabilità, o analizzare comportamenti con configurazioni diciamo "simil produzione".

Anche in questo caso Podman ci può aiutare permettendo l'installazione di un ambiente più sofisticato e più simile ad una configurazione di produzione. Di seguito ecco alcuni semplici comandi che ci permetteranno di eseguire in locale un cluster di due (o più) nodi di Keycloak in alta affidabilità con persistenza su un databse PostgresQL.

## Deploy

Prima di tutto ci serve creare una rete interna, i due nodi di Keycloak infatti condividono le sessioni web attraverso un cluster Infinispan che si forma automaticamente mediante un sistema di auto-discovery che cerca tutti i nodi in esecuzione sulla stessa rete. Inoltre entrambi i nodi devono dialogare con il DB Postgres che contiene le configurazioni di Keycloak.

cominciamo quindi con il comando:

> podman network create kc-network

Naturalmente potete chiamare la rete come meglio credete.
Provate ad eseguire il comando

> podman network ls

dovreste avre un output del tipo:

> NETWORK ID    NAME          DRIVER
> 2f259bab93aa  podman        bridge
> 6a5df8616cf3  kc-network  bridge

Ora scarichiamo le immagini che ci servono:

> podman pull docker.io/library/postgres:latest
>
> podman pull quay.io/keycloak/keycloak:latest

Per primo faremo partire il Database Postgres ma prima di lanciare il comando per eseguirlo creiamo anche un volume per la persistenza dei dati che andranno sul database:

> podman volume create pgdata

verificate che il volume sia stato correttamente creato e verificate anche il mount point:

> podman volume inspect pgdata

a questo punto possiamo eseguire il database, usate:

> podman run -d --name pgsql --net kc-network -p 5432:5432 -v pgdata:/var/lib/postresql/data -e POSTGRES_USER=dbuser -e POSTGRES_PASSWORD=change_me -e POSTGRES_DB=keycloakDB postgres

**\[NOTA\]** Potreste anche tralasciare di creare un volume e quindi anche eliminare l'opzione **-V** nel comando precedente, ricordatevi però che in quel caso se fermate il pod che contiene il DB e poi lo eseguite di nuovo perderete le configurazioni salvate nella sessione precedente (che potrebbe anche essere un caso d'uso voluto).

Sempre al fine di controllare che tutto stia funzionando correttamente potreste usare il comando: 

> podman exec pgsql env

che vi ritornerà una lista di variabili tra cui dovreste vedere anche quelle inserite nel comando precedente.

Altro comando che vi tornerà utile è

> podman network inspect kc-network

in cui potrete ricavare l'indirizzo IP assegnato da Podman al container che esegue PostgresQL (e che utilizzerete più avanti).

## \[Opzionale\] Download and Start pgAdmin

Questa parte non è necessaria, la riporto perché in generale può far comodo, in ambiente di sviluppo, ispezionare il database delle configurazioni di Keycloak. Se volete provare aggiunmgete questi comandi, altrimenti saltate a [`Build and Run Keycloak`](#build).

> podman pull docker.io/dpage/pgadmin4:latest

> podman run -d --name pgadmin -p 8082:80 --net kc-network -e PGADMIN_DEFAULT_EMAIL="dbuser@example.com" -e PGADMIN_DEFAULT_PASSWORD="change_me" pgadmin4

Potete adesso usare la web console di Postgres usando il browser ed aprendo la pagina [localhost:8082](http://localhost:8082)

## Build and Run Keycloak

Con riferimento a questa [guida](https://www.keycloak.org/server/containers) la prima cosa che dobbiamo fare è costruire una immagine locale personalizzata di Keycloak a partire da quella ufficiale scaricata all'inizio.

Questo passaggio è molto importante perché ci permette di aggiungere al Keycloak base tutti i plugin, provider ed ogni qualunque altro tipo di personalizzazione di cui necessita il nostro serve Keycloak.

Per questo esempio io ho allegato un *Dockerfile* con cui specifico i parametri di connessione al DB e preparo dei certificati self-signed per le connessioni HTTPS. In particolare fate attenzione alla variabile **KC_HOSTNAME** dove dovrete specificare l'IP che Podman ha assegnato alla rete interna (e che avete letto nell'output del comando di inspect di cui sopra).

> podman build . -t kc

Ora tutto è pronto per lanciare le due (o più) istanze di Keycloak:

> podman run -d --rm --name kc-1 --net kc-network -p 8443:8443 -p 9000:9000 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=change_me kc start --optimized --hostname=localhost

nella seconda istanza (o successive) dovrete fare attenzione a modificare il nome del pod (in realtà il nome potete anche ometterlo, sarà podman ad assegnare un nome random al pod) ed anche le porte esposte sul vostro PC, se non le cambiate infatti podman le troverà occupate e non riuscira a far partire il pod.

> podman run -d --rm --name **kc-2** --net kc-network -p 8543:8443 -p 9100:9000 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=change_me kc start --optimized --hostname=localhost

## \[Opzionale\] Aggiungiamo un bilanciatore nell'architettura

