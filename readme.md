#DOCKER

## CRIANDO UMA IMAGEM

-   Crair um arquivo chamado `dockerfile` (sem extenção).

-   Dizer para o docker que a versão do dockerfile para utilizar é a 1 - mais recente. Ele automaticamente verifica se você está utilizando a versão mais recente.
    `# syntax=docker/dockerfile:1`

-   Dizer qual imagem base é para ser usada em nosso aplicativo.
    `FROM node:12.18.1`

-   Específicar variável de ambiente NODE_ENV (não obrigatório).
    `ENV NODE_ENV=production`

-   Para tornar as coisas mais fáceis ao executar o resto de nossos comandos, vamos criar um diretório de trabalho. Isso instrui o Docker a usar esse caminho como o local padrão para todos os comandos subsequentes. Dessa forma, não precisamos digitar os caminhos completos dos arquivos, mas podemos usar caminhos relativos com base no diretório de trabalho.
    `WORKDIR /app`

-   Agora iremos copiar os arquivos de configuração de dependências do projeto (package.json, package.lock) para dentro de nossa imagem.
    Note que o terceiro(último) argumento diz o local para onde os mesmos devem ser copiados.
    `COPY ["package.json", "yarn.lock", "./"]`

-   Agora precisamos instalar as depenências listadas em nosso package.json para dentro de nossa imagem.
    `RUN yarn install --production`

-   Agora iremos copiar nossos arquivos do projeto para dentro de nossa imagem. Copiará tudo do diretório atual para nossa imagem.
    `COPY . .`

-   Agora precisamos dizer o que fazer quando noss imagem for executada. Vamos dizer para executar o servidor.
    `CMD ["node", "server.js"]`

### A sintáxe deve ficar da seguinte forma:

`# syntax=docker/dockerfile:1 FROM node:12-alpine ENV NODE_ENV=production WORKDIR /app COPY package.json yarn.lock ./ RUN yarn install --production COPY . . CMD ["node", "src/index.js"]`

## CRIANDO UM ARQUIVO .DOCKERIGNORE

Para que o docker não copie a pasta node_modules mesmo após a mesma já existir devido ao comando yarn install. Devemos dizer ao docker para o mesmo ignorar-la.

Dentro do arquivo .dockerignore, é só dizer qual pasta é para o mesmo ignorar, que no nosso caso é a pasta node_modules.

_arquivo .dockerignore_
`node_modules`

## CONSTRUINDO A IMAGEM

Para construir a imagem, você deve estar com o CMD dentro da pasta dom seu projeto que tenha o arquivo **dockerfile**.

Execute o comando `docker build --tag node-docker`.

A flag `--tag` é onde você pode definir uma tag para o a imagem. Pode ser a versão, simples tag... Caso não seja passada nenhuma tag, o docker irá adicionar a tag "lasted" à imagem.

**node-docker** é o nome da nossa imagem.

## COMANDOS ÚTEIS (na verdade todos são)

(no lugar do id pode também ser o nome do container)

-   Listar as imagens: `docker images`
-   Listar os containers em execução: `docker ps`
-   Listar todos os containers: `docker ps --all`
-   Deletar um container: `docker rm -f <id-do-container>`
-   Parar um container: `docker stop <id-do-container>`
-   Iniciar um container: `docker start <id-do-container>`
-   Reiniciar um container: `docker restart <id-do-container>`

## CRIANDO UM CONTAINER

Um contêiner é um processo normal do sistema operacional, exceto que esse processo é isolado e tem seu próprio sistema de arquivos, sua própria rede e sua própria árvore de processo isolada separada do host.

Para executar um container utilizamos o comando `docker run` seguido do nome da imagem. (mais para frente iremos nos aprofundar mais sobre esse comando)
`docker run node-docker`

Caso seu aplicativo seja uma API, por exemplo, não conseguiremos acessar a mesma pois não definimos em qual porta ela estará executando em nosso container.

Para publicar uma porta para nosso contêiner, usaremos a flag `--publish` ( -p para abreviar) no comando `docker run`. O formato do comando --publish é [host port]:[container port]. Portanto, se quiséssemos expor a porta 8000 dentro do contêiner para a porta 3000 fora do contêiner, passaríamos 3000:8000 para a flag `--publish`.

O comando final ficará dessa forma:
`docker run -p 8000:8000 node-docker`

Caso queira rodar em segundo plano, podemos passar a flag `-d`, o comando completo ficará mais ou menos assim: `docker run -dp 8000:8000 node-docker`

---

## CRIANDO VOLUMES(PERSISTINDO O DADOS DO BANCO DE DADOS)

Quando trabalhamos com bancos de dados no docker, ao desligarmos nosso container, perdemos também todos os dados que estavam amarzenados no banco de dados do mesmo. Para que isso não ocorra, existem os **volumes**, que são parte da memória que são compartilhadas de nosso computador com o container. Permitindo assim que os dados persistam.

Em vez de baixar o MongoDB, instalar, configurar e executar o banco de dados Mongo como um serviço, podemos usar a imagem oficial do Docker para MongoDB e executá-lo em um contêiner.

-   Antes de executarmos o MongoDB em um contêiner, queremos criar alguns volumes que o Docker pode gerenciar para armazenar nossos dados e configuração persistentes.

`docker volume create mongodb`
`docker volume create mongodb_config`

-   Agora, iremos criar uma network (uma rede) para que nossos containers possam se comunicar entre si. A rede é chamada de rede de ponte definida pelo usuário e nos oferece um bom serviço de pesquisa de DNS que podemos usar ao criar nossa string de conexão

`docker network create monogdb`

-   Agora poderemos iniciar o MongoDB em um container e anexar os volumes e a rede que criamos acima.
    `docker run -it --rm -d -v mongodb:/data/db -v mongodb_config:/data/configdb -p 27017:27017 --network mongodb --name mongodb mongo`

(o `--name` também pode ser alterado para `--network-alias`)

-   Agora, vamos executar nosso contêiner. Mas, desta vez, precisaremos definir a variável de ambiente CONNECTIONSTRING para que nosso aplicativo saiba qual string de conexão usar para acessar o banco de dados. Faremos isso bem no coman do docker run.

`docker run \ -it --rm -d \ --network mongodb \ --name rest-server \ -p 8000:8000 \ -e CONNECTIONSTRING=mongodb://mongodb:27017/yoda_notes \ node-docker`

## FACILITANDO TUDO COM O DOCKER COMPOSER

Para não precisar ter todo esse trabalho na hora da criação ou compartilhamento de um projeto, por exemplo. Existe o docker-compose, um arquivo onde listamos tudo isso e que podemos compartilhar e rodar tudo com apenas um comando.

-   Primeiro criamos o arquivo **docker-compose.yml**

Nele, inserimos os seguintes dados:
``version: '3.8'

services:
notes:
build:
context: .
ports:

-   8000:8000
-   9229:9229
    environment:
-   SERVER_PORT=8000
-   CONNECTIONSTRING=mongodb://mongo:27017/notes
    volumes:
-   ./:/app
    command: npm run debug

mongo:
image: mongo:4.2.8
ports:

-   27017:27017
    volumes:
-   mongodb:/data/db
-   mongodb_config:/data/configdb
    volumes:
    mongodb:
    mongodb_config:``

Para criar tudo, é so executar o comando `docker-compose up -d` dentro da raiz do projeto.

Você pode ver os logs utilizando o comando `docker-compose -f` e caso queria ver apenas de um container específico, é só por o nome dele a frente: `docker-compose -f <nome>`

---

Para mais informações sobre o composer: <https://docs.docker.com/get-started/08_using_compose/>

Jarod Cavalcante - 2021
