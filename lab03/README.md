# Lab03 - Criação de containers Docker
> [Voltar](../README.md)

Nesse laboratório, iremos criar containers Docker, utilizando o programa de linha de comando `docker`, através de uma VM Ubuntu Server 20.04, criada no `Lab01` utilizando VirtualBox.

## 1. Preparação do ambiente para laboratório
Para esse laboratório, precisaremos realizar a instalação do Docker no Ubuntu 20.04, caso não esteja instalado:
```shell
# Utilizaremos o gerenciador de pacotes snap para instalar o Docker
sudo snap install docker
```

Para utilizarmos o docker sem usuário root, precisamos adicionar o grupo docker ao usuário ubuntu:
```shell
# Adicionar o grupo docker no sistema
sudo addgroup --system docker
# Adiciona o usuário ubuntu no grupo docker
sudo adduser $USER docker
# Logar utilizando o novo grupo
newgrp docker
# Desabilitar e reabilitar o Docker Daemon
sudo snap disable docker
sudo snap enable docker
```

Baixar a imagem do Linux Alpine que utilizaremos para criação do nosso RootFS:
```shell
docker pull alpine
```

Acessar a pasta home do usuário e criar a pasta lab03:
```shell
cd
mkdir lab03
cd lab03
```

### 1.1. Obter o IP da VM na rede do host
```shell
ip addr show
```
Exemplo de resposta:
```shell
ubuntu@servidor01:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:75:74:17 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 85976sec preferred_lft 85976sec
    inet6 fe80::a00:27ff:fe75:7417/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:21:0d:a7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.102/24 brd 192.168.56.255 scope global dynamic enp0s8
       valid_lft 477sec preferred_lft 477sec
    inet6 fe80::a00:27ff:fe21:da7/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:a3:b8:56:d3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:a3ff:feb8:56d3/64 scope link
       valid_lft forever preferred_lft forever
```
No exemplo acima, a interface `enp0s8`, que possuí o MAC Address `08:00:27:21:0d:a7`, correspondente ao `Adaptador 2` da VM no VirtualBox configurado no modo `Placa de rede exclusiva de hospedeiro (host-only)`, possuí o IP `192.168.56.102`. Utilizaremos esse IP para demonstrações abaixo. Substitua o IP `192.168.56.102` pelo IP correspondente à sua VM.

Utilizar o comando a seguir para obter seu IP:
```shell
ip route | grep 192.168.56 | awk '{print $9}'
```

## 2. Docker CLI
Criaremos um container Docker utilizando o comando `docker`.

### 2.1. Terminal 1
Abrir um novo terminal e executar os comandos a seguir.
```shell
# Entrar no diretório do lab03
cd lab03
# Listar os comandos disponíveis no docker
docker --help
# Listar os containers em execução
docker ps
# Baixar a imagem containertools/hello-world do Docker Hub (https://hub.docker.com/r/containertools/hello-world)
docker pull containertools/hello-world
# Listar as imagens baixadas
docker images
# Criar um container a partir da imagem containertools/hello-world
docker run -it -p 8888:8080 --name hello-world containertools/hello-world
# `-it`: Executar o container em modo interativo
# `-p 8888:8080`: Expor a porta 8080 do container na porta de rede 8888 do host
# `--name hello-world`: Nomear o container de hello-world
# `containertools/hello-world`: Imagem a partir da qual o container será criado
```

Acessar o navegador na URL [http://192.168.56.102:8888](http://192.168.56.102:8888). Observe a contagem de PIDs.

### 2.2. Terminal 2
Abrir um segundo terminal e executar os comandos a seguir.
```shell
# Entrar no diretório do lab03
cd lab03
# Listar os containers em execução
docker ps
# Entrar no container hello-world
docker exec -it hello-world sh
# Recarregue a página http://192.168.56.102:8888 e observe que a contagem de PIDs aumentou.
# Listar os arquivos dentro do container
ls -la
# Verificar o sistema operacional atual (Alpine) 
cat /etc/os-release
# Verificar o código fonte da página HTML que acessamos no navegador
cat templates/index.html
# Sair do container
exit
```

### 2.3. Terminal 1
Voltar no terminal 1, e interromper a execução do container, pressionando `Ctrl+C`.
```shell
# Criar um container em background com o parâmetro `-d`
docker run -it -d -p 8888:8080 --name hello-world containertools/hello-world
# Observe que ocorreu um erro, indicando o nome `hello-world` já está em uso por outro container. Para isso, precisaremos deletar o container anterior que criamos
# Listar os containers em execução
docker ps
# Observe que o container anteriormente criado não aparece. Isso ocorre devido ao comando docker ps mostrar apenas os containers em execução.
# Para listarmos todos os containers criados, independente se estão em execução ou não, iremos utilizar o parâmetro `-a`
docker ps -a
# Deletar o container hello-world. 
# Opcionalmente podemos especificar o ID do container obtido no `docker ps`, ao invés de seu nome, em qualquer comando que precisarmos mencionar um container.
docker rm hello-world
# Criar um container em background
docker run -itd -p 8888:8080 --name hello-world containertools/hello-world
# Listar o container criado
docker ps
# Obter os logs do container
docker logs hello-world
# Parar o container hello-world
docker stop hello-world
# Listar todos os containers
docker ps -a
# Observe o status do container hello-world, indicando que foi finalizado (Exited...).
# Startar o container hello-world
docker start hello-world 
# Listar todos os containers
docker ps -a
# Observe o status do container hello-world, indicando que está em operação (Up...)
# Caso o container não pare utilizando o comando `docker stop`, podemos forçar sua interrupção, utilizando o comando `docker kill`.
docker kill hello-world
# Listar todos os containers
docker ps -a
# Observe o status do container hello-world, indicando que foi finalizado (Exited...)
# Startar o container hello-world
docker start hello-world
# Listar todos os containers
docker ps -a
# Observe o tempo indicado no status UP
# Restartar o container hello-world
docker restart hello-world
# Listar todos os containers
docker ps -a
# Observe o tempo indicado no status UP
# Verificar o consumo de recursos do container
docker stats hello-world
# Recarregue a página http://192.168.56.102:8888 e observe o consumo dos recursos mudarem. Pressione ``Ctrl+C` para interromper o comando.
# Obter os eventos Docker gerados na última hora
docker events --since=1h
# Observe que foram registrados os eventos das operações de start, stop, kill, entre outros. Pressione ``Ctrl+C` para interromper o comando.
# Conectar na entrada e saída padrão do container hello-world
docker attach hello-world
# Recarregue a página http://192.168.56.102:8888 e observe que o log de acesso ao container que conectamos, aparece no terminal. Isso ocorre devido a estarmos conectados à saída padrão do container.
# Pressione Ctrl+C e observe que interrompemos a execução do serviço dentro do container. Isso ocorre devido a estarmos conectados à entrada padrão do container, que ao pressionarmos Ctrl+C, enviamos um sinal de interrupção (SIGINT) para o serviço em execução.
# Listar todos os containers
docker ps -a
# Startar o container hello-world
docker start hello-world
# Entrar no container hello-world
docker exec -it hello-world sh
# Criar um arquivo de teste
echo "Isso é um teste" > /tmp/teste.txt
# Sair do container
exit
# Visualizar as alterações realizadas no container a partir da imagem base
docker diff hello-world
# Obter os detalhes do container hello-world. Nesses detalhes são mostrados pontos de montagem, informações de rede, overlayfs, variáveis de ambiente, estado do container, etc.
docker inspect hello-world
# Parar todos os containers em execução 
docker stop $(docker ps -a -q)
# Remover todos os containers
docker container prune
# Pressione y seguido de Enter para remover todos os containers
# Remover todas as imagens não utilizadas
docker image prune -a
# Pressione y seguido de Enter para remover todas as imagens não utilizadas
# De maneira alternativa, podemos remover imagens utilizando o comando `docker rmi <imagem>`. Exemplo: `docker rmi containertools/hello-world`
```

## 3. Dockerfile
Criaremos uma imagem de container de um servidor de arquivo, utilizando Dockerfile. Para isso, utilizaremos o comando `nano Dockerfile`, para editar o arquivo `Dockerfile` conforme indicado no decorrer do laboratório.

Após alterar o arquivo `Dockerfile`, para salvar, seguir a seguinte sequência de comandos:
- Pressione Ctrl+X
- Pressione Y
- Pressione Enter

```shell
# Baixar a image alpine
docker pull alpine
# Editar o arquivo Dockerfile
nano Dockerfile
```

Alterar o arquivo Dockerfile:
```Dockerfile
FROM alpine

RUN apk add python3

CMD ["python3", "-m", "http.server"]
```

A instrução `FROM alpine` indica que nossa imagem será construída a partir da imagem [alpine](https://hub.docker.com/_/alpine). A instrução `RUN apk add python3` indica para rodar o comando `apk add python3`, que realiza a instalação do Python 3 na imagem do container. A instrução `CMD ["python3", "-m", "http.server"]` indica que o container será iniciado com o comando `python3 -m http.server` quando for criado.

Executar os comandos a seguir:
```shell
# Contruir a imagem do container
# `-t file-server` indica que a imagem será salva com o nome `file-server` 
# `.` indica para construir a imagem a partir da pasta onde estamos (utilizar o comando `pwd` para verificar em qual pasta você está)
docker build -t file-server .
# Criar um container com a imagem `file-server` que acabamos de construir.
# `-rm` indica que o container será destruído após o final da sua execução
docker run -it --rm -p 8000:8000 file-server
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Observe que é possível visualizar todos os arquivos dentro do nosso container, porém para nosso servidor de arquivos, queremos limitar o acesso apenas a uma pasta específica do nosso container. Para isso, criaremos a pasta `/files` utilizando a instrução `WORKDIR /files` na imagem do nosso container, que ficará dedicada ao servidor de arquivos. Essa instrução realiza a criação da pasta indicada caso não exista, e entra nessa pasta, fazendo com que as instruções seguintes sejam executadas a partir dessa pasta indicada. 

Editar o arquivo Dockerfile:
```shell
# Pressione Ctrl+C para interromper a execução do container
nano Dockerfile
```

Alterar para:
```Dockerfile
FROM alpine

RUN apk add python3

WORKDIR /files

CMD ["python3", "-m", "http.server"]
```

Executar os comandos a seguir:
```shell
# Contruir a nova imagem do container
docker build -t file-server .
# Criar um novo container com a imagem atualizada
docker run -it --rm -p 8000:8000 file-server
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Observe que agora não aparecem arquivos no nosso servidor, dado que criamos uma pasta vazia na imagem do container. Agora criaremos um arquivo dentro da pasta `/files` na imagem do container:

Editar o arquivo Dockerfile:
```shell
# Pressione Ctrl+C para interromper a execução do container
nano Dockerfile
```

Alterar para:
```Dockerfile
FROM alpine

RUN apk add python3

WORKDIR /files
RUN echo "Arquivo de teste" > teste.txt

CMD ["python3", "-m", "http.server"]
```
Com a instrução `RUN echo "Arquivo de teste" > teste.txt`, executamos o comando `echo "Arquivo de teste" > teste.txt`, que cria o arquivo `teste.txt` com o conteúdo `Arquivo de teste` dentro da pasta onde estamos (`/files`).

Executar os comandos a seguir:
```shell
# Contruir a nova imagem do container
docker build -t file-server .
# Criar um novo container com a imagem atualizada
docker run -it --rm -p 8000:8000 file-server
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Clicar no arquivo `teste.txt`. Observe que o conteúdo do arquivo corresponde ao que indicamos no arquivo Dockerfile. 

Executar os comandos a seguir:
```shell
# Pressione Ctrl+C para interromper a execução do container
# Criar pasta host
mkdir host
# Criar o arquivo host/teste-host.txt
echo "Arquivo de teste do host" > host/teste-host.txt
# Criar um novo container com a imagem atualizada
# `-v "$PWD/host:/files/host"` indica que montaremos a pasta host que criamos, na pasta /files/host dentro do container, para que possamos compartilhar arquivos do host para nosso servidor de arquivos
docker run -it --rm -p 8000:8000 -v "$PWD/host:/files/host" file-server
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Clicar em `host/` e em seguida clicar em `teste-host.txt`. Observe que o conteúdo do arquivo corresponde ao que criamos a partir do host. Pressione `Ctrl+C` para interromper a execução do container.

## 4. Docker Compose
Criaremos o arquivo `docker-compose.yml` para especificarmos o serviço `file-server`, contendo toda a configuração necessária para execução do nosso servidor de arquivos.

Editar o arquivo `docker-compose.yml`:
```shell
nano docker-compose.yaml
```
Inserir o seguinte conteúdo:
```yaml
version: '3'
services:
  file-server:
    image: file-server
    volumes:
      - ./host:/files/host
    ports:
      - 8000:8000
```

A opção `version: '3'` indica que utilizaremos a versão 3 da especificação docker compose. A opção `services:` indica que especificaremos serviços, para execução de containers. Dentro de `services:` temos o service `file-server`, que representa o container do servidor de arquivos. A opção `image: file-server` dentro de `file-server` indica que utilizaremos a image `file-server` construída anteriormente, como imagem base do nosso container. A opção `volumes:`, com o valor `- ./host:/files/host`, indica a montagem da pasta `./host` no host em `/files/host` no container. A opção `ports:`, com o valor `- 8000:8000`, indica que o serviço `file-server` irá utilizar a expor a porta 8000 no host, a partir da porta 8000 do container.

Executar o seguinte comando para construção e execução do container utilizando Docker Compose:
```shell
docker-compose up -d
```
O parâmetro `-d` indica para executar o container em background.

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Explorar os arquivos listados pelo servidor.

Incluir no arquivo `docker-compose.yml` o novo volume `vol-host` gerido pelo Docker:
```shell
nano docker-compose.yaml
```

Alterar para:
```yaml
version: '3'
services:
  file-server:
    image: file-server
    volumes:
      - ./host:/files/host
      - vol-host:/files/vol-host
    ports:
      - 8000:8000
volumes:
  vol-host:
      driver: local
```

Com a opção `volumes:` no final do arquivo, podemos especificar quais volumes deverão ser criados, e seus drivers correspondentes. Neste caso utilizaremos o driver `local`, que indica que o volume será criado em um diretório local no host. Outros drivers podem ser utilizados, para utilização de volumes remotos, como `NFS`, `CIFS`, `Amazon S3`, etc. Adicionamos o valor `- vol-host:/files/vol-host` em volumes do service `file-server`, indicando a montagem do volume `vol-host` criado localmente em `/files/vol-host` no container.

Executar os comandos a seguir:
```shell
# Atualizar o service `file-server` com as novas configurações, recriando o container
docker-compose up -d
# Entrar no container
docker-compose exec file-server sh
# Criar um arquivo no container
echo "Arquivo que não está em um volume" > arquivo.txt
# Entrar no volume `vol-host`
cd vol-host
# Criar um arquivo no volume montado
echo "Arquivo de teste no volume vol-host" > vol.txt
# Sair do container
exit
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Clicar em `arquivo.txt` e verificar seu conteúdo. Voltar para [http://192.168.56.102:8000](http://192.168.56.102:8000). Clicar em `vol-host/`, em seguida clicar em `vol.txt` e verificar seu conteúdo.

Executar os comandos a seguir:
```shell
# Listar os services
docker-compose ps
# Parar o service `file-server`
docker-compose stop file-server
# Remover o service `file-server`, destruindo o container
docker-compose rm file-server
# Pressione y seguido de Enter para remover o container file-server
# Validar a remoção do container
docker-compose ps
# Criar um novo service `file-server`
docker-compose up -d
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Observe que o arquivo `arquivo.txt` não aparece na listagem de arquivos. Como ele foi criado em um diretório que não existia um volume montado, quando excluímos o container através do comando `docker-compose rm file-server`, todos os arquivos que não estavam em volumes também foram removidos. O arquivo `teste.txt` se manteve na listagem de arquivos, devido a ele estar contido na imagem base do container. Observe que os arquivos criados em volumes, se mantiveram na listagem de arquivos.

Executar os comandos a seguir:
```shell
# Visualizar logs do service file-server
docker-compose logs file-server
# Criar arquivo que jogaremos no volume `vol-host`
echo "Arquivo que jogaremos no volume via cp" > cp.txt
# Listar os containers
docker-compose ps
# Obter o nome do container, através da coluna `Name`. Utilizaremo de exemplo o nome do container `lab03_file-server_1`. Utilizar o nome que aparecer para você.
# Copiar o arquivo `cp.txt` para o caminho `/files/vol-host/cp.txt` dentro do service `file-server`
# docker cp <caminho do arquivo dentro do host> <nome do container>:<caminho do arquivo dentro do container>
docker cp cp.txt lab03_file-server_1:/files/vol-host/cp.txt
# Deletar o arquivo `cp.txt` do host
rm cp.txt
```

Acessar [http://192.168.56.102:8000](http://192.168.56.102:8000). Clicar em `vol-host/` e em seguida clicar em `cp.txt`. Observe que o arquivo `cp.txt` foi copiado para dentro do volume `vol-host`.

Para destruir todos os services, e redes associadas, executar o seguinte comando:
```shell
docker-compose down
```