# PostgreSQL Workshop 1.0

 Egyszerűség kedvért legyen valamilyen központi könyvtár, amire van jogosultságunk és tegyük egy változóba, hogy később arra lehessen hivatkozni (lehet akár /home/felhasznalonev/projekt is):
 ```bash
 export PROJECTS_HOME="/srv/projects"
 # letrehozas, jogosultsag beallitas
 sudo install -d -o ${USER} -g ${USER} -m 2775 ${PROJECTS_HOME}
 ```

## Git hogyan alapok

1. Távoli repository leszedése "klónozása" helyi mappába:

    ```bash
    cd ${PROJECTS_HOME}
    git clone https://github.com/remedioshu/psqlwshop
    cd psqlwshop
    ```

2. Helyi repository frissítése a távoli "origin" -ből, amennyiben az frissebb (Fontos: abban a mappastruktúrában kell kiadni a git parancsokat, amelyik "rekurzív szülőmappában" a .git található és azon belül a "config", mely tartalmazza a beállításokat -pl a remote origin címét-)

    ```bash
    cd ${PROJECTS_HOME}/psqlwshop
    git pull
    ```

3. Változáskezelési napló megtekintése 

    ```bash
    git log
    # pontos diff változásokkal együtt:
    git log -p
    ```

4. Változáskezelési azonosítóhoz tartozó összes változás megtekintése (COMMIT_ID értéke az első n > 4 egyedi karakter az adott git-en belül)

    ```bash
    git show COMMIT_ID
    ```

## Docker telepítése, alapok
1. legfrissebb APT vagy YUM esetén (utóbbinál addons vagy epel kellhet hozzá)

    ```bash
    yum install docker-engine docker-compose || apt install docker.io docker-compose
    ```
    **Fontos**: *CentOS/Redhat környezetben a /tmp noexec -el került csatolásra. A docker-compose működéséhez viszont szükség van olyan TMPDIR=/tmp beállításra, amely csatolási ponton van exec jog!*
    ```bash
    # amennyiben a ${PROJECTS_HOME} csatolási pontján van exec, akkor:
    sudo install -d -m 1777 ${PROJECTS_HOME}/tmp
    export TMPDIR=${PROJECTS_HOME}/tmp
    ```

2. Docker image réteg leszedése (csak letöltés/frissítés):
    ```bash
    # alapértelmezett docker-registry a docker.io ("docker info" -val meg lehet nézni)
    docker pull postgres:10
    docker pull dpage/pgadmin4:4.28
    docker pull nicolaka/netshoot:latest

    # meghatározott registry-ből (amennyiben a registry.host.fqdn:port publikus vagy már "docker login registry.host.fqdn:port" azonosítás megtörtént!)
    docker pull registry.host.fqdn:port/postgres:10
    ```

3. Docker image réteg letöltése (ha még nincs letöltve) és új "ideiglenes" konténer réteg létrehozása, majd elindítása (start):
    ```bash
    # a -it interaktív terminál, --rm opció a konténer réteg automatikus törlése a konténerben futó process kilépése után, pl jelen esetben exit
    docker run -it --rm postgres:10 bash
    ```

4. Csak a futó konténerek listázása:
    ```bash
    docker ps
    ```

5. Összes konténer listázása:
    ```bash
    docker ps -a
    ```

6. Már létező (létrehozott) konténer indítása/leállítása:
    ```bash
    # docker ps -a kiírja, hogy mi a neve és vagy id-je
    # pl: 
    CONTAINER_NAME_vagy_ID=psqltest
    docker start ${CONTAINER_NAME_vagy_ID}
    docker stop ${CONTAINER_NAME_vagy_ID}
    ```

7. Dockerrel kapcsolatos hibakeresések
    ```bash
    # NAME_vagy_ID értéke lehet akár konténer, hálózat, volume és egyéb is.
    # pl 
    NAME_vagy_ID=psqltest; CONTAINER_NAME_vagy_ID=psqltest
    docker inspect ${NAME_vagy_ID}
    # adott kontener logjai (STDERR és STDOUT)
    docker logs ${CONTAINER_NAME_vagy_ID}
    # adott kontener-ben futo processek listaja
    docker container top ${CONTAINER_NAME_vagy_ID}
    # adott kontener-ben elerheto parancs futtatasa 
    #    -it interaktiv, igy tovabbi parancsok futtathatoak belul
    docker exec -it ${CONTAINER_NAME_vagy_ID} bash
    #    vagy epp nem interaktiv es az eredmenyt a host-ra kapjuk
    docker exec ${CONTAINER_NAME_vagy_ID} ls -al
    ```

## Postgresql telepítése vagy Docker indítása
1. Csomagkezelős telepítés, mely alapértelmezetten a /etc/postgresql

    ```bash
    yum install postgresql-server || apt install postgresql
    ```

2. A PostgreSQL hivatalos Docker hub oldalán (https://hub.docker.com/_/postgres) elérhető leírás alapján, de kiegészíte:
    ```bash
    # network=host esetén nincs NAT, egy az egyben hoston fut -ha jogosult és a default 5432 port szabad
    # -e környezeti_változó=érték, melyet a Dockerfile alapján feldolgoz az docker image indításkor
    # fontos, hogy itt nincs tartós tároló így a konténer törlésekor az adatbázis is elveszik!
    docker run -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres --network=host postgres:10
    ```
    
3. A PostgreSQL közvetlen a host-on, a DB adatok és beállításai egy tartós tárolóba (helyi mappába) kerülnek. Amennyiben a konténer törlésre, majd újra létrehozása kerül az adatok megmaradnak:
    ```bash
    # a mappanak uresnek kell lennie, hogy a postgresql initdb lefuthasson
    mkdir -p ${PROJECTS_HOME}/psqltest/pgdata
    # -v static_path_to_dir:container_dir
    docker run --name psqltest -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -v "${PROJECTS_HOME}/psqltest/pgdata:/var/lib/postgresql/data" --network=host postgres:10 &
    ```

4. A PostgreSQL elindítása docker-compose segítségével, amely tartalmaz öszetettebb beállításokat is (hasonlóan egy OpenShift/Kubernetes környezethez). Beletettem egy tartós tároló és önálló hálózat kezelést is
    ```bash
    # letrehozas es/vagy elinditas ami hatasara mindent a YAML allomany nevevel prefix-el meg
    docker-compose -f ${PROJECTS_HOME}/psqlwshop/psql1/psql.yml up &
    # tartos tarak listaja ahol psql1_pgdata1 lesz
    docker volume ls
    # megnezni pl, hogy hol tarolja
    docker inspect psql1_pgdata1
    # halozatok listaja
    docker network ls

    # torlés es leallitas (tartos tarat alapbol nem torol)
    docker-compose -f psql.yml down
    ```

## A PostgreSQL CLI használata
1. A kliens verziójának kiírása
    ```bash    
    psql --version
    ```

2. A kliens verziójának kiírása docker-en belül:
    ```bash    
    # letrehoz egy uj kontenert, ami az adott image-n alapul majd torli azt    
    docker run --rm postgres:10 psql --version
    ```

2. Hálózati kapcsolódás a docker-ben a testnet hálózaton futó PostgreSQL adatbázishoz:
    ```bash    
    # docker network ls -el lehet tudni melyik halozatban szeretnenk inditani (lehet host is akar)
    docker run -it --rm --network=psql1_testnet postgres:10 psql -h psql1_psql1_1 -p 5432 -U postgres
    ```
    docker run -it --name psql1_client --network=host postgres:10 psql -h 15432 -U postgres


# Hasznos linkek egyes témával kapcsolatban:
# Git:
1. https://nvie.com/posts/a-successful-git-branching-model/

# Docker:
1. https://docs.docker.com/engine/reference/commandline/docker/

# PostgreSQL
1. https://wiki.postgresql.org/wiki/Converting_from_other_Databases_to_PostgreSQL#Oracle
2. https://www.postgresqltutorial.com/postgresql-show-tables/
