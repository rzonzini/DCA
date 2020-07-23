# O QUE É ISTO
- Material produzido pelo [Jérôme Petazzoni](https://twitter.com/jpetazzo)
- A versão online está [aqui](https://container.training/swarm-selfpaced.yml.html#1)
- Para usar, abra o [swarm-selfpaced.yml.html](https://github.com/rzonzini/DCA/blob/orchestration/swarm-selfpaced.yml.html) no navegador

# RESUMO

## 1. Antes do capítulo *Global schedule*

#### 1. Precisamos ter um cluster rodando, claro!

```
docker swarm init --advertise-addr $(hostname -i)`
docker swarm join-token worker`
docker swarm join-token manager`
```

#### 2. Clonando a apliação de exemplo

```
git clone https://github.com/jpetazzo/container.training
```

#### 3. Criando um REGISTRY

```
docker service create --name registry --publish 5000:5000 registry
```

#### 4. Colocando as imagens do projeto no REGISTRY

```
cd ~/container.training/dockercoins
export REGISTRY=127.0.0.1:5000
export TAG=v0.1
for SERVICE in hasher rng webui worker; do \
  docker build -t $REGISTRY/$SERVICE:$TAG ./$SERVICE; \
  docker push $REGISTRY/$SERVICE; \
done
```

#### 5. Criando uma rede para a stack da aplicação do curso exemplo

```
docker network create --driver overlay dockercoins
```

#### 6. Rodando a aplicação

```
docker service create --network dockercoins --name redis redis
for SERVICE in hasher rng webui worker; do \
  docker service create --network dockercoins --detach=true \
  --name $SERVICE $REGISTRY/$SERVICE:$TAG; \
done
docker service update webui --publish-add 8000:80
```

#### 7. Escalando a aplicação *webui*

```
docker service update --replicas 10
```

## 2. Global scheduling (p. 192)

#### 1. Escalando o servico *rng* para ter uma task em execução em cada node (p. 199)

- Removendo o serviço criado em [Rodando a aplicação](#6-rodando-a-aplica%C3%A7%C3%A3o)
```
docker service rm rng
```
+ Recriando o serviço com o *global scheduling*
```
docker service create --name rng --network dockercoins --mode global $REGISTRY/rng:$TAG
```

#### 2. Removendo todos os serviços (p. 201)
```
docker service ls -q | xargs docker service rm
```

## 3. Swarm Stack (p. 212)

#### 1. Subindo o serviço do REGISTRY como uma stack (p. 217)
```
docker stack deploy --compose-file registry.yml registry
```

#### 2. Usando o *compose* para fazer o build e subir as imagens para o REGISTRY (p. 221)
```
cd ./container.training/stacks
docker-compose -f dockercoins.yml build
docker-compose -f dockercoins.yml push
```

#### 3. Fazendo o deploy da stack
```
docker stack deploy --compose-file dockercoins.yml dockercoins
```

## 4. Parte 2 (p. 232)

#### 1. Montando as coisas para continuar (p. 234)
```
docker swarm init --advertise-addr $(hostname -i)
TOKEN=$(docker swarm join-token -q manager)
NODE1IP=<ip_do_node1>
for N in <lista_dos_ips_dos_nodes>; do
  DOCKER_HOST=tcp://$N:2375 docker swarm join --token $TOKEN $NODE1IP:2377
done
DOCKER_HOST=tcp://<ip_node_worker>:2375 docker swarm join --token $(docker swarm join-token -q worker) $NODE1IP:2377
git clone https://github.com/jpetazzo/container.training
cd container.training/stacks
docker stack deploy --compose-file registry.yml registry
docker-compose -f dockercoins.yml build
docker-compose -f dockercoins.yml push
docker stack deploy --compose-file dockercoins.yml dockercoins
```

## 5. Breaking into an overlay network (p. 239)
Requer o ambiente [montado](#1-montando-as-coisas-para-continuar-p-234)

#### 1. Criando um container "faz nada" (p. 234)
```
docker service create --network dockercoins_default --name debug --constraint node.hostname==$HOSTNAME alpine sleep 1000000000
```

#### 2. Entrando no container criado pelo serviço no passo [anterior](#1-criando-um-container-faz-nada-p-234) (p. 242)
```
docker exec -it $(docker ps -q --filter label=com.docker.swarm.service.name=debug) sh
```

#### 3. Sobre VIP e DNSRR (p. 243)

1. Instalando algumas ferramentas
```
apk add --update curl apache2-utils drill
```
2. `drill rng` vai retornar uma saída igual ao do *dig* mostrando o endereço VIP do serviço *rng*;
3. O serviço pode ser publicado usando VIP ou DNSRR:
   - com VIP, temos um IP virtual e um balanceador de carga baseado no IPVS;
   - com o DNSRR, o nome do serviço nos endereços IP nos nodes onde existem suas tarefas; e
   - Use `docker service create --endpoint-mode [VIP|DNSRR]` para ecolher entre os dois modos.
4. `drill tasks.rng` retorna o endereço IP de todos os container onde o *rng* está sendo executado;

## 6. Securing overlay networks (p. 263)

#### 1. Criando redes (p. 265)

- criando uma rede insegura
```
docker network create insecure --driver overlay --attachable
```

- criando uma rede segura
```
docker network create secure --opt encrypted --driver overlay --attachable
```

#### 2. Subindo webservice (p. 266)
```
docker service create --name web --network secure --network insecure --constraint node.hostname!=node1 nginx
```

#### 3. Sniff o tráfego HTTP (p. 266)
```
docker run --net host nicolaka/netshoot ngrep -tpd eth0 HTTP
```
- no Play-With-Docker, para ver o trafego do host com a internet efetivar o sniff do acesso a uma página na web, por exemplo `curl www.pudim.com.br`, o comando acima deve escutar a interface *eth1*. A *eth0*, é o tráfego do entre os nós do cluster.
- outro bom bizu é parar a aplicação *worker* do serviço *dockercoins*: `docker service update dockercoins_worker --replicas=0`
- sniffando a comunicação da rede *insecure*, criada em [Criando redes]{#1-criando-redes-p-265}, com `docker run --rm --net insecure nicolaka/netshoot curl web` é possível ver fragmentos HTTP
- sniffando a comunicação da rede *secure*, criada em [Criando redes]{#1-criando-redes-p-265}, com `docker run --rm --net secure nicolaka/netshoot curl web` nenhum fragmento HTTP é exibido, aparecendo somente os '#' referentes a cada pacote que está atravesando a interface

## 7. Updating services (p. 273)

#### 1. Atualizando um único serviço com *service update* (p. 275)
```
cd ~/container.training/stacks/dockercoins
export REGISTRY=127.0.0.1:5000
export TAG=v0.2
IMAGE=$REGISTRY/dockercoins_webui:$TAG
docker build -t $IMAGE webui/
docker push $IMAGE
docker service update dockercoins_webui --image $IMAGE
```

#### 2. Atualizando serviços com *stack update* (p. 276)

- fazendo uma alteração no código

```
sed -i "s/15px/50px/" dockercoins/webui/files/index.html
```
- criando uma nova versão com essa alteração e atializando a *stack*

```
export TAG=v0.2
docker-compose -f dockercoins.yml build
docker-compose -f dockercoins.yml push
docker stack deploy -c dockercoins.yml dockercoins
```

## 8. Rolling updates (p. 282)

#### 1. Executando a atualização (p. 283)

- legal usar `docker events` para acompanhar as ações do swarm
- `--force` pode ser usado pra trocar containers que não sofreram mudança na configuração

```
docker service scale dockercoins_hasher=7
docker service update --image 127.0.0.1:5000/hasher:v0.1 dockercoins_hasher
```

#### 2. Alterando a política de atualização (p. 283)

```
docker service update --update-parallelism 2 --update-max-failure-ratio .25 dockercoins_hasher
```

- a alteração da política de atualização também pode ser feita no arquivo Compose com a chave `update_config`, dentro da chave `deploy`

#### 3. Dando última forma na atualização (p. 286)

```
docker service rollback dockercoins_webui
```

- pode ser executado no arquivo Compose, usando a flag `--rollback` em `service update` ou `docker service rollback`
- cada `docker service update` gera uma nova definição que pode ser observada em `PreviousSpec` no detalhamento do servico (`docker service inspect`)

## 9. Health checks e rollback automático (p. 290)

## 10. Feramentas para debugar o SwarmKit (p. 301)

## 11. Secrets management and encryption at rest (p. 313)

#### 1. Criar o *secret* (p. 316)

- Usando uma senha fraca mas é didático para mostrar a sintaxe do comando
```
echo love | docker secret create hackme -
```

- Uma boa maneira de criar a senha é usando um valor aleatório assossiado a um algorítomo de criptografia, por exemplo, o base64
```
base64 /dev/random | head -c16 | docker secret create arewesecureyet -
```

#### 2. Usando o *secret* (p. 318)

```
docker service create --secret hackme --secret arewesecureyet --name dummyservice --constraint node.hostname==$HOSTNAME alpine sleep 10000000000
```

#### 3. Acessando o *secret* (p. 319)

- O *secret* passado para o *service* fica armazenado no diretório `/run/secrets`. Então, acessando o container criado e consultando o conteúdo desse diretório, podemos ver os arquivos que tem o *secret* criado como seu conteúdo.

```
docker exec -it $(docker ps -q --filter label=com.docker.swarm.service.name=dummyservice) grep -r . /run/secrets
```

#### 4. Alterando o *secret* de um serviço (p. 321)

- *secrets* podem ser mapeados com nomes diferentes. No exmplo abaixo, `--secret-add source=arewesecureyet,target=hackme`, mapeia o secret *hackme* para usar o mesmo conteúdo do secret *arewesecureyet*

```
docker service update dummyservice --secret-rm hackme
docker service update dummyservice --secret-add source=arewesecureyet,target=hackme
docker exec -it $(docker ps -q --filter label=com.docker.swarm.service.name=dummyservice) grep -r . /run/secrets
/run/secrets/arewesecureyet:hUMqA5X4Du8dL+xJ
/run/secrets/hackme:hUMqA5X4Du8dL+xJ
```

##### 5. Trancando o cluster (p. 325)

- A senha gerada é requerida para "destrancar" o cluster e permitir a execução executar comandos de administração do swarm
- O serviço do *dockerd* deve ser reiniciado para efetivar a mudança, por exemplo, com `systemctl restart docker`

```
docker swarm update --autolock=true
Swarm updated.
To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-flfY4EUJYSOlZoVu+6OT3qJ+xzTrZMZ/oF6PL0MfkTg

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```

- A senha gerada é solicitada para destrancar o cluster

```
docker swarm unlock
```

- Para recriar a senha mas atenção: se existir algum node "bloqueado" quando a senho for recriada, esse node não consegirá ingressar no cluster!

```
docker swarm unlock-key --rotate
```

- Para recuperar a senha, desde que exista, ao menos um node manager "destravado", a partir dele:

```
docker swarm unlock-key -q
```

- Para desbloquar o node manager permanentemente:

```
docker swarm update --autolock=false
```

- Sempre é possível fazer o node deixar de ser um manager e assumir a função de worker: `docker node demote <nome_do_node>` e, depois de `docker swarm leave` no node agora worker, em algum manager `docker node rm <nome_do_node>`

## 12. Menos privilégio (p. 333)

## 13. Log centralizado (p. 343)

## 14. Configurando o ELK para armazenar logs (p. 347)

> 1. **ElasticSearch**: armazena e indexa os logs
> 2. **LogStash**: recebe os logs
> 3. **Kibana**: web UI

#### 1. Sobindo os serviços da stack (p. 350) 

```
docker network create --driver overlay logging
docker service create --network logging --name elasticsearch elasticsearch:2.4
docker service create --network logging --name kibana --publish 5601:5601 -e ELASTICSEARCH_URL=http://elasticsearch:9200 kibana:4.6
docker service create --network logging --name logstash -p 12201:12201/udp logstash:2.4 -e "$(cat ~/container.training/elk/logstash.conf)"
```

- vendo o heartbeat

```
docker service logs logstash -f
```

- O jpetazzo fez um (docker-compose)[https://github.com/jpetazzo/container.training/blob/master/stacks/elk.yml] com essa stack que pode ser consultado na (página 355)[https://container.training/swarm-selfpaced.yml.html#355]

#### 2. Verificando o funcionamento (p. 356)

- Vendo os logs do *logstash*

```
docker service logs --follow --tail 1 logstash
```

- Em paralelo, em uma nova sessão porque a anterior está travada com o logs, subi um container qualquer usando o driver `--log-driver gelf` *gelf* e passando o socket `--log-opt gelf-address=udp://127.0.0.1:12201` onde o *logstash* está recebendo os dados, os logs enviados podem ser vistos na janela anterior 

```
docker run --log-driver gelf --log-opt gelf-address=udp://127.0.0.1:12201 --rm alpine echo hello
```

#### 3. Enviando os logs de um serviço (p. 358)

- se `docker service logs --follow --tail 1 logstash` ainda estiver aberto, os log do serviço podem ser vistos

```
docker service create --name alpine --log-driver gelf --log-opt gelf-address=udp://127.0.0.1:12201 alpine echo hello
```

- esse serviço vai ficar em loop, sendo restado infinitamente, até que uma condição de restart seja definida. As condições possíveis são: *none*, *any* e *on-error*; ainda é possível usar oputros parâmetros como: *--restart-delay*, *--restart-max-attempts* e *--restart-window*

```
docker service update alpine --restart-condition none
```

- atualizar serviços preexistentes para enviar logs para o *logstash*

```
docker service update dockercoins_rng --log-driver gelf --log-opt gelf-address=udp://127.0.0.1:12201
```

------------------------------------------------------------------------------------------------------


# MarkMaker

General principles:

- each slides deck is described in a YAML manifest;
- the YAML manifest lists a number of Markdown files
  that compose the slides deck;
- a Python script "compiles" the YAML manifest into
  a HTML file;
- that HTML file can be displayed in your browser
  (you don't need to host it), or you can publish it
  (along with a few static assets) if you want.


## Getting started

Look at the YAML file corresponding to the deck that
you want to edit. The format should be self-explanatory.

*I (Jérôme) am still in the process of fine-tuning that
format. Once I settle for something, I will add better
documentation.*

Make changes in the YAML file, and/or in the referenced
Markdown files. If you have never used Remark before:

- use `---` to separate slides,
- use `.foo[bla]` if you want `bla` to have CSS class `foo`,
- define (or edit) CSS classes in [workshop.css](workshop.css).

After making changes, run `./build.sh once`; it will
compile each `foo.yml` file into `foo.yml.html`.

You can also run `./build.sh forever`: it will monitor the current
directory and rebuild slides automatically when files are modified.

If you have problems running `./build.sh` (because of
Python dependencies or whatever),
you can also run `docker-compose up` in this directory.
It will start the `./build.sh forever` script in a container.
It will also start a web server exposing the slides
(but the slides should also work if you load them from your
local filesystem).


## Publishing pipeline

Each time we push to `master`, a webhook pings
[Netlify](https://www.netlify.com/), which will pull
the repo, build the slides (by running `build.sh once`),
and publish them to http://container.training/.

Pull requests are automatically deployed to testing
subdomains. I had no idea that I would ever say this
about a static page hosting service, but it is seriously awesome. ⚡️💥


## Extra bells and whistles

You can run `./slidechecker foo.yml.html` to check for
missing images and show the number of slides in that deck.
It requires `phantomjs` to be installed. It takes some
time to run so it is not yet integrated with the publishing
pipeline.
