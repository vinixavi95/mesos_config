# mesos_config
Cluster mesos configurado localmente, gerenciado pelo mesos rodando um zookeeper, mesos-master, mesos_slave, mesos_slave-2, marathon and chronos.

# Mesos Docker Containers

## Running Mesos locally
Esse setup é ideal para desenvolvimento e aprendizagem, existem limitações quando se trata de usar o Marathon para rodar containers Docker.

###docker-compose.yaml reference
```
version: '3'
services:

  zookeeper:
    image: mesoscloud/zookeeper:3.4.6-ubuntu-14.04
    hostname: "zookeeper"
    container_name: zookeeper
    ports:
      - "2181:2181"
      - "2888:2888"
      - "3888:3888"

  mesos-master:
    image: datastrophic/mesos-master:1.1.0
    hostname: "mesos-master"
    container_name: master
    privileged: true
    environment:
      - MESOS_HOSTNAME=mesos-master
      - MESOS_CLUSTER=SMACK
      - MESOS_QUORUM=1
      - MESOS_ZK=zk://zookeeper:2181/mesos
      - MESOS_LOG_DIR=/tmp/mesos/logs
    links:
      - zookeeper
    ports:
      - "5050:5050"

  mesos-slave:
    image: datastrophic/mesos-slave:1.1.0
    hostname: "mesos-slave"
    container_name: slave
    privileged: true
    environment:
      - MESOS_HOSTNAME=mesos-slave
      - MESOS_PORT=5151
      - MESOS_MASTER=zk://zookeeper:2181/mesos
    links:
      - zookeeper
      - mesos-master
    ports:
      - "5151:5151"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  marathon:
    image: datastrophic/marathon:1.3.6
    hostname: "marathon"
    container_name: marathon
    environment:
      - MARATHON_HOSTNAME=marathon
      - MARATHON_MASTER=zk://zookeeper:2181/mesos
      - MARATHON_ZK=zk://zookeeper:2181/marathon
    links:
      - zookeeper
      - mesos-master
    ports:
      - "8080:8080"

  chronos:
    image: datastrophic/chronos:mesos-1.1.0-chronos-3.0.1
    hostname: "chronos"
    container_name: chronos
    environment:
      - CHRONOS_HTTP_PORT=4400
      - CHRONOS_MASTER=zk://zookeeper:2181/mesos
      - CHRONOS_ZK_HOSTS=zookeeper:2181
    links:
      - zookeeper
      - mesos-master
    ports:
      - "4400:4400"

Esse arquivo de configuração deve ser colado no arquivo docker-compose-yml para rodar o cluster localmente

###Rodando o Mesos cluster
Cluster com zookeeper, master, agents, marathon e chronos:

Levantar o cluster:
      $ sudo su
      $ docker-compose -p mesos up

Recomeçar e levantar o cluster:
      $ docker-compose -p mesos up -d --force-recreate


Depois de digitar o comando as interfaces dicarão disponíveis em:

* Mesos Master [http://localhost:5050](http://mesos-master:5050)
* Marathon [http://localhost:8080](http://marathon:8080)
* Chronos [http://localhost:4400](http://chronos:4400)

Parar o cluster: 
      $ docker-compose -p mesos down

Parar o cluster e remover os containers:
      $ docker-compose -p mesos down && docker-compose -p mesos rm -f
            
###Usando multiplos agentes localmente
É possível utilizar multiplos mesos agentes, para isso no documento `docker-compose.yml` você precisa duplicar o bloco de código do mesos agente, alterando e remapeando as portas para evitar conflitos. Você pode checar o arquivo [docker-compose-bare-mesos.yml](docker-compose-bare-mesos.yml) 
para ter uma referência.

###Limitações do setup local
No cluster Mesos de produção, a melhor prática é executar o Chronos via Marathon para fornecer garantias de alta disponibilidade.
Isso pode ser feito postando a configuração apropriada do Marathon para o aplicativo Chronos:
  
      curl -XPOST 'http://marathon:8080/v2/apps' -H 'Content-Type: application/json' -d '{
        "id": "chronos",
        "container": {
          "type": "DOCKER",
          "docker": {
              "network": "BRIDGE",
              "privileged":true,
              "image": "datastrophic/chronos:mesos-1.1.0-chronos-3.0.1",
              "parameters": [
                   { "key": "env", "value": "CHRONOS_HTTP_PORT=4400" },
                   { "key": "env", "value": "CHRONOS_MASTER=zk://zookeeper:2181/mesos" },
                   { "key": "env", "value": "CHRONOS_ZK_HOSTS=zookeeper:2181"}
              ],
              "portMappings": [
                { "containerPort": 4400 }
              ]
          }
        },
        "cpus": 1,
        "mem": 512,
        "instances": 1
      }'
  
No entanto, esta configuração não funcionará localmente porque o Marathon não oferece suporte a tipos de rede personalizados e a nenhum outro tipo de rede
O contêiner do Chronos NÃO conseguirá alcançar o tratador e vincular o Chronos ao endereço roteável.

