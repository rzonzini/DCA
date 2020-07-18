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
+ com VIP, temos um IP virtual e um balanceador de carga baseado no IPVS;
+ com o DNSRR, o nome do serviço nos endereços IP nos nodes onde existem suas tarefas; e
+ Use `docker service create --endpoint-mode [VIP|DNSRR]` para ecolher entre os dois modos.
4. `drill tasks.rng` retorna o endereço IP de todos os container onde o *rng* está sendo executado;



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
