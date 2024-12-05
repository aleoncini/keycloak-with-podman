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

## \[Opzionale\] Download and Start pgAdmin

Questa parte non è necessaria, la riporto perché in generale può far comodo, in ambiente di sviluppo, ispezionare il database delle configurazioni di Keycloak. Se volete provare aggiunmgete questi comandi, altrimenti saltate a [`Build and Run Keycloak`](#build).

## Build and Run Keycloak