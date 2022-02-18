# Docker-swarm-with-portainer
Docker Swarm with portainer

# Docker swarm - Portainer

## Requeremientos

- 3 manager como minimo
- 2 node como minimo
- Docker engine https://docs.docker.com/engine/install/
- Docker compose v1 o v2 https://docs.docker.com/compose/install/#alternative-install-options
## Instalar docker y docker compose en cada maquina sea nodo o manager 

- Desinstalar cualquier version venga por defecto en la maquina de linux  

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
````

- Actualice el índice de paquetes apt e instale paquetes para permitir que apt use un repositorio a través de HTTPS:

```bash
sudo apt-get update
```

```bash
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
- Agregue la clave GPG oficial de Docker:

```bash
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

- Utilice el siguiente comando para configurar el repositorio estable.
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- Actualice el índice del paquete apt e instale la última versión de Docker Engine y containerd, o vaya al siguiente paso para instalar una versión específica:

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

## Instalar docker compose version 2

- Ejecute este comando para descargar la versión estable actual de Docker Compose:
```bash
 sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

- Aplicar permisos ejecutables al binario:

```bash
sudo chmod +x /usr/local/bin/docker-compose
````
- Pruebe la instalación.
```bash
docker-compose --version
```

Resultado Docker Compose version v2.2.3

## Docker swarm 

![Image text](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)
-  Inicializar swarm manager principal la ip es la corresponde a la maquina hemos escogido como manager lider obtener podemos ejecutar el comando ip -a
```bash
sudo docker swarm init --advertise-addr  xxx.xxx.xxx.xxx
```

- como resultado para agregar un trabajador a este enjambre del manager principal, ejecutamos el siguiente comando ejemplo dentro los workes o nodos:
```bash
sudo docker swarm join --token SWMTKN-1-2s4gmtierl28je7xdzjb4xmkd6pqol3cdxvc6v5xmdkbxqbv5j-axenr2kxroddysfyb94mp3c58 xxx.xxx.xxx.xxx:2377
```

- Agregar otro manager ejecutamos el siguiente comando en el manager principal para generar el token con que los otros manager se van unir:
```bash 
sudo docker swarm join-token manager
````
- como resultado para agregar un trabajador a este enjambre, ejecute el siguiente comando ejemplo dentro los managers :
```bash
sudo docker swarm join --token SWMTKN-1-2s4gmtierl28je7xdzjb4xmkd6pqol3cdxvc6v5xmdkbxqbv5j-39e5u7snxqjhbcjipt9z5uwo5 xxx.xxx.xxx.xxx:2377
```

- Para listar nodos y manager ejecutamos el siguiente comando

```bash
sudo docker node ls 
```
```
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
v99v60tphabrijq1ervytcq52 *   manager1   Ready     Active         Leader           20.10.12
0hzzug83g0nd7dpmk24v64hp7     manager2   Ready     Active         Reachable        20.10.12
hwt5kzw9qahyn118kj7ozh0u4     manager3   Ready     Active         Reachable        20.10.12
l67ozg632t09c181beyfxx5xb     node1      Ready     Active                          20.10.12
lga3jyawwb7scvxtmyrx0xyf5     node2      Ready     Active                          20.10.12
```
## Portainer con swarm

### Portainer 

- Crear el archivo portainer.yml dentro manager1

```
version: '3.2'

services:
  agent:
    image: portainer/agent:2.11.1
    environment:
      # REQUIRED: Should be equal to the service name prefixed by "tasks." when
      # deployed inside an overlay network
      AGENT_CLUSTER_ADDR: tasks.agent
      # AGENT_PORT: 9001
      # LOG_LEVEL: debug
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:2.11.1
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent_network:
    driver: overlay

volumes:
  portainer_data:
```

### Portainer con NFS  (Recomienda para persistencia de los datos)

- Requeremiento instalar nfs-common en todos los managers, nodos o workes

```bash
sudo apt install -y nfs-common
````
- Testear conexión en mi caso mi servidor NFs esta ip 192.168.56.11  
```bash
sudo showmount -e xxx.xxx.xxx.xxx
```

- Crear el archivo portainer.yml dentro manager1

```
version: '3.2'

services:
  agent:
    image: portainer/agent:2.11.1
    environment:
      # REQUIRED: Should be equal to the service name prefixed by "tasks." when
      # deployed inside an overlay network
      AGENT_CLUSTER_ADDR: tasks.agent
      # AGENT_PORT: 9001
      # LOG_LEVEL: debug
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:2.11.1
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9000:9000"
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent_network:
    driver: overlay

volumes:
  portainer_data:
    driver_opts:
      type: "nfs4"
      o: "addr=xxx.xxx.xxx.xxx,rw"
      device: ":/mnt/Nas/portainer"

```

- Ejecutamos el docker portainer 

**Nota. Ejecutar un stack en el clúster como servicio con el siguiente comando se hace para todos**

```bash
sudo docker stack deploy --compose-file=portainer.yml portainer 
```
- Accedemos via web http://xxx.xxx.xxx.xxx:9000

- Desconectar el stack: es necesario utilizar docker stack rm con el nombre del stack.
```bash
sudo docker stack rm portainer
```

- Desconectar el registro servicio: hay que recurrir al comando docker service rm con el nombre de servicio, en este caso, 

```bash
sudo docker service rm portainer
```
