# Lab02 - Criação de um container Linux
> [Voltar](../README.md)

Nesse laboratório, vamos criar um container Linux, através de uma VM Ubuntu Server 20.04. Podemos utilizar o VirtualBox para criação da VM, ou de maneira alternativa, podemos utilizar um ambiente [Ubuntu 20.04 no Killercoda](https://killercoda.com/playgrounds/scenario/ubuntu). 

## Preparação do ambiente para laboratório
Para esse laboratório, precisaremos realizar a instalação do Docker no Ubuntu 20.04, caso não esteja instalado:
```shell
# Utilizaremos o gerenciador de pacotes snap para instalar o Docker
sudo snap install docker
```

`OPCIONAL` Para utilizarmos o docker sem usuário root, precisamos adicionar o grupo docker ao usuário ubuntu:
```shell
# Criaremos um grupo docker caso não exista
sudo groupadd docker
# Adicionaremos o usuário ubuntu ao grupo docker
sudo usermod -aG docker ubuntu
# Reiniciar o sistema aplicando o seguinte comando
sudo init 6
```

Trocar para o usuário root:
```shell
sudo su
```

Baixar a imagem do Linux Alpine que utilizaremos para criação do nosso RootFS:
```shell
docker pull alpine
```

Acessar a pasta home do usuário e criar a pasta lab02:
```shell
cd
mkdir lab02
cd lab02
```

## OverlayFS
Criaremos a estrutura de diretórios do OverlayFS dentro da pasta `fs`, populando com o RootFS Alpine, e montando o OverlayFS no diretório `fs/merged`, para iniciarmos a criação do nosso container. Quando mencionar `No host`, deveremos executar os comandos na máquina host, enquanto `No container` deveremos executar os comandos dentro do container (utilizando `chroot`).
### No host
```shell
# Criar a estrutura de diretórios do OverlayFS
mkdir -p fs/{lower,upper,work,merged}
# Exportar a imagem alpine para o diretório fs/lower
docker export $(docker create alpine) | tar -C fs/lower -xf -
# Montar o OverlayFS (Irá aparecer algo como "mount: none mounted on /root/lab02/fs/merged", significando que a montagem foi realizada com sucesso. O `none` significa que não existe uma partição física correspondente para a montagem, já que estamos realizando a montagem a partir dos diretórios do OverlayFS.)
mount -vt overlay -o lowerdir=./fs/lower,upperdir=./fs/upper,workdir=./fs/work none ./fs/merged
# Verificar o sistema operacional atual (Ubuntu)
cat /etc/os-release
# Entrar no sistema operacional virtual
chroot fs/merged /bin/sh
```
### No container
```shell
# Verificar o sistema operacional atual (Alpine)
cat /etc/os-release
# Sair do container
exit
```

## Volume
Agora criaremos um volume para montarmos no nosso container, permitindo persistir os dados do container em uma pasta local do host.
### No host
```shell
# Criar a pasta volume, para persistencia dos dados do container
mkdir volume 
# Criar um arquivo de teste, com o conteúdo `Meu arquivo de teste`
echo "Meu arquivo de teste" > volume/lab02.txt
# Criar uma pasta volume dentro do diretório fs/merged para realizarmos a montagem da pasta volume do host dentro do container
mkdir ./fs/merged/volume
# Montar o volume no container
mount -v --bind -o ro ./volume ./fs/merged/volume
# Entrar no container
chroot fs/merged /bin/sh
```
### No container
```shell
# Entrar na pasta /volume
cd /volume
# Tentar adicionar no final do arquivo de teste criado anteriormente, a frase `Um olá de dentro do container!`
echo "Um olá de dentro do container!" >> lab02.txt
# Observe que não foi possível alterar o arquivo de teste, pois ao realizarmos a montagem do volume, utilizamos o parâmetro `-o ro`, que indica que o volume é somente leitura. Vamos sair do container e montar novamente o volume, com permissão de escrita.
exit
```
### No host
```shell
# Desmontar o volume do container
umount ./fs/merged/volume
# Montar o volume no container, com permissão de leitura e escrita
mount -v --bind -o rw ./volume ./fs/merged/volume
# Entrar no container
chroot fs/merged /bin/sh
```
### No container
```shell
# Entrar na pasta /volume
cd /volume
# Adicionar a frase `Um olá de dentro do container!` no final do arquivo de teste
echo "Um olá de dentro do container!" >> lab02.txt
# Sair do container
exit
```
### No host
```shell
# Obter o conteúdo do arquivo de teste
cat ./volume/lab02.txt
# Observe que o conteúdo do arquivo contém as alterações que realizamos dentro do container
```

## Namespace
Agora criaremos um namespace para o container, permitindo que o container seja executado em um ambiente isolado.
### No host
```shell
# Verificar qual o ID do processo atual (PID) no shell que estamos
echo $$
# Listar os processos que enxergamos no host
ps aux
# Entrar no container
chroot fs/merged /bin/sh
```
### No container
```shell
# Montar o diretório /proc para que possamos listar os processos de dentro do container
mount -vt proc proc /proc
# Verificar qual o ID do processo atual (PID) no shell que estamos dentro do container
echo $$
# Listar os processos que enxergamos no container
ps aux
# Observe que não existe isolamento de processos, pois o container não está em um ambiente isolado. Para isso, vamos sair do container e criar um isolamento para nosso container, através de namespaces.
exit
```

### No host
```shell
# Criar um namespace para o container, utilizando o comando unshare, isolando pontos de montagem (--mount) e processos (--pid), realizando a montagem do sistema de arquivos proc criado dentro do container (fs/merged/proc). Com o parâmetro --fork criamos uma novo processo, que será o responsável por executar o container utilizando o comando chroot.
unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh
```
### No container
```shell
# Verificar qual o ID do processo atual (PID) no shell que estamos dentro do container
echo $$
# Listar os processos que enxergamos no container
ps aux
# Observe que o isolamento funcionou, não sendo mais possível enxergar os processos do host
exit
```

## Network
Agora isolaremos a camada de rede do container, fazendo com que o container acesse outras redes através da rede virtual que criaremos dedicada ao container.
```shell
# Entrar no container
unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh
```
### No container
```shell
# Verificar as interfaces de rede disponíveis ao container
ip link
# Observe que conseguimos acessar as interfaces de rede disponíveis no host, dado que não isolamos a camada de network
exit
```

### No host
#### Criação do switch virtual
Criaremos um switch virtual (Linux Bridge), para prover conectividade ao nosso container.
```shell
# Criar o namespace de rede `cnt` para nosso container
ip netns add cnt
# Criar o switch virtual `meu-switch`
ip link add meu-switch type bridge
# Adicionar o IP 10.123.231.1 para o switch virtual
ip addr add 10.123.231.1/24 brd + dev meu-switch
# Ligar o switch virtual
ip link set meu-switch up
```
#### Criação das interfaces de rede virtuais
```shell
# Criar a interface de rede virtual `veth-cnt` do container, conectado à interface de rede virtual `meu-sw-veth-1` do switch virtual
ip link add veth-cnt type veth peer name meu-sw-veth-1
# Adicionar a interface de rede virtual `veth-cnt` ao namespace de rede `cnt`, para que nosso container isolado possa acessar essa interface
ip link set veth-cnt netns cnt
# Adicionar a interface de rede virtual `meu-sw-veth-1` ao switch virtual
ip link set meu-sw-veth-1 master meu-switch
# Ligar a interface de rede `meu-sw-veth-1`
ip link set meu-sw-veth-1 up
# Adicionar o IP 10.123.231.2 para a interface de rede virtual `veth-cnt` do container
ip -n cnt addr add 10.123.231.2/24 dev veth-cnt
# Ligar a interface localhost do container
ip -n cnt link set lo up
# Ligar a interface `veth-cnt` do container
ip -n cnt link set veth-cnt up
# Adicionar o `meu-switch` como rota padrão para o container, permitindo o encaminhamento de pacotes de rede do container para outras redes
ip -n cnt route add default via 10.123.231.1 dev veth-cnt
# Configurar o SNAT (Source NAT) para pacotes provenientes da rede 10.123.231.0/24, cujo o destino sejam outras interfaces de rede distintas à bridge `meu-switch`
# `iptables -t nat -I POSTROUTING 1`: Cria uma regra de SNAT momento antes de realizar o roteamento. Adiciona essa regra na posição 1 da tabela nat.
# `-s 10.123.231.0/24`: A regra é aplicada apenas para pacotes cujo a origem seja um IP da subnet 10.123.231.0/24.
# `! -o meu-switch`: A regra é aplicada apenas para pacotes cujo a interface de saída não seja `meu-switch`.
iptables -t nat -I POSTROUTING 1 -s 10.123.231.0/24 ! -o meu-switch -j MASQUERADE
# Configurar a filtragem de pacotes, permitindo que pacotes que chegam da interface bridge `meu-switch`, sejam encaminhados para outras interfaces diferentes da `meu-switch`
iptables -I FORWARD -i meu-switch ! -o meu-switch -j ACCEPT
# Configurar a filtragem de pacotes, permitindo que pacotes que chegam da interface bridge `meu-switch`, sejam encaminhados para a própria interface `meu-switch`
iptables -I FORWARD -i meu-switch -o meu-switch -j ACCEPT
# Configurar a filtragem de pacotes, permitindo que outras máquinas troquem pacotes de redes em conexões estabelecidas com a rede 10.123.231.0/24
iptables -I FORWARD -o meu-switch -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

#### Montar arquivo de configuração DNS para o container
```shell
# Criar um diretório para armazenar o arquivo de configuração DNS
mkdir -p netns/cnt
# Criar o arquivo de configuração DNS para o container
echo nameserver 8.8.8.8 > netns/cnt/resolv.conf
# Montar o arquivo de configuração do Host para o container
mount --bind ./netns/cnt/resolv.conf ./fs/merged/etc/resolv.conf
# Desabilitar o roteamento via Kernel Linux
sysctl -w net.ipv4.ip_forward=0
# Entrar no container, aplicando o namespace de rede `cnt` ao container
ip netns exec cnt unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh
```
### No container
```shell
# Verificar as interfaces de rede disponíveis ao container
ip link
# Observe que não conseguimos mais visualizar as interfaces de rede do hosts, apenas as interfaces de rede configuradas no namespace
# Verificar o acesso do container ao IP 1.1.1.1, através do comando `ping`
ping 1.1.1.1
# Observe que não conseguimos acessar o IP 1.1.1.1, ou seja, não recebemos resposta dessa máquina 1.1.1.1. Para interromper o ping, pressionar `Ctrl+C`, finalizando o comando ping em execução. Isso ocorre por conta de termos desabilitado o roteamento de pacotes de rede através do Kernel Linux
exit
```
### No host
```shell
# Habilitar o roteamento via Kernel Linux
sysctl -w net.ipv4.ip_forward=1
# Entrar no container
ip netns exec cnt unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh
```
### No container
```shell
# Verificar o acesso do container ao IP 1.1.1.1, através do comando `ping`. Para interromper o ping, pressionar `Ctrl+C`.
ping 1.1.1.1
# Verificar o acesso do container ao www.google.com, através do comando `ping`. Para interromper o ping, pressionar `Ctrl+C`.
ping www.google.com
# Observe que conseguimos acessar o IP 1.1.1.1 e o domínio www.google.com
exit
```
Caso o ping para 1.1.1.1 e www.google.com não funcione, pode ser que na rede onde está sendo executado o lab, exista um bloqueio de firewall para o protocolo ICMP utilizado durante o ping. De maneira alternativa, substitua os pings realizados nos passos anteriores, utilizando o seguinte comando `ping` para testar a conectividade:
```shell
ping 192.168.56.1
```
Repita os passos a partir do comando `# Desabilitar o roteamento via Kernel Linux`.

### Para persistir a configuração net.ipv4.ip_forward
```shell
# Salvar a configuração net.ipv4.ip_forward no arquivo /etc/sysctl.conf
sudo nano /etc/sysctl.conf
```
- Adicione no final do arquivo a seguinte linha: `net.ipv4.ip_forward=1`
- Pressione Ctrl+X
- Pressione Y
- Pressione Enter

## Cgroup
Agora limitaremos a criação de processos no container, através da configuração do cgroup.
### No host
```shell
# Criar um diretório cnt/lab02 no cgroup pids. O diretório /sys/fs/cgroup é um diretório de arquivos que contém os grupos de controle para os processos Linux. Quando criamos uma nova pasta dentro de /sys/fs/cgroup/<cgroup>, o Kernel Linux cria um grupo de controle dedicado, onde podemos especificar os processos que serão geridos por ele.
mkdir -p /sys/fs/cgroup/pids/cnt/lab02
# Limitar a quantidade de processos que podem ser criados dentro do container em 3
echo 3 > /sys/fs/cgroup/pids/cnt/lab02/pids.max
```
### Terminal 2
Em em segundo terminal, realizar os seguintes comandos:
```shell
# Trocar para o usuário root
sudo su
# Acessar a pasta do container no host
cd && cd lab02
# Entrar no container, através do shell (/bin/sh) com o parâmetro `-v`
ip netns exec cnt unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh -v
```
### Terminal 3
Em em terceiro terminal, realizar os seguintes comandos:
```shell
# Trocar para o usuário root
sudo su
# Acessar a pasta do container no host
cd && cd lab02
# Entrar no container, através do shell (/bin/sh) com o parâmetro `-v`
ip netns exec cnt unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh -v
```

### No host
```shell
# Criar variável de ambiente `CNT_PID` contendo como o valor a lista de processos `sh` com parâmetro `-v`. Nesse caso, serão os dois shells que estão rodando dentro do container.
CNT_PID=$(lsns -t pid | grep "sh -v" | awk '{print $4}')
# Configurar o cgroup criado anteriormente, para controlar os processos em execução dentro do container
echo "$CNT_PID" > /sys/fs/cgroup/pids/cnt/lab02/tasks
```

### Terminal 1
```shell
# Criar um novo processo com o comando sleep, mantendo ele em execução por 30 segundos
sleep 30
```
### Terminal 2
```shell
# Criar um novo processo com o comando sleep, mantendo ele em execução por 30 segundos
sleep 30
# Observe que ao tentar criar um novo processo, o Kernel Linux não permite que o processo seja criado, dado que extrapolou o limite imposto pelo cgroup configurado. No caso temos 2 processos `sh -v` e 1 processo `sleep 30`, e estamos tentando executar o quarto processo `sleep 30`.
```

### No host
```shell
# Remover o limite imposto anteriormente
echo max > /sys/fs/cgroup/pids/cnt/lab02/pids.max
```
### Terminal 1
```shell
# Criar um novo processo com o comando sleep, mantendo ele em execução por 30 segundos
sleep 30
```
### Terminal 2
```shell
# Criar um novo processo com o comando sleep, mantendo ele em execução por 30 segundos
sleep 30
# Observe que agora que removemos o limite imposto anteriormente, conseguimos criar o quarto processo `sleep 30` dentro do container.
```

## Limpeza
Para removermos as configurações realizadas nesse laboratório, utilizar os seguintes comandos:
```shell
# Desmontar recursivamente o diretório fs/merged. Todo e qualquer diretório montado dentro de fs/merged, será desmontado automaticamente
umount -R ./fs/merged
# Remover configurações iptables
iptables -t nat -D POSTROUTING -s 10.123.231.0/24 ! -o meu-switch -j MASQUERADE
iptables -D FORWARD -i meu-switch ! -o meu-switch -j ACCEPT
iptables -D FORWARD -i meu-switch -o meu-switch -j ACCEPT
iptables -D FORWARD -o meu-switch -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# Remover interfaces de rede virtuais
ip link del meu-sw-veth-1
ip link del meu-switch
# Remover namespace de rede
ip netns del cnt
# Remover cgroups criados
$(cd /sys/fs/cgroup/pids && rmdir -p cnt/lab02)
```


## Kit montagem
Seguem comandos utilizados para a montagem do laboratório futuramente:
```shell
# Acesso ao diretório do laboratório
sudo su
cd && cd lab02
# Montagens para o container
mount -vt overlay -o lowerdir=./fs/lower,upperdir=./fs/upper,workdir=./fs/work none ./fs/merged
mount -vt proc proc ./fs/merged/proc
mount -v --bind -o rw ./volume ./fs/merged/volume
mount --bind ./netns/cnt/resolv.conf ./fs/merged/etc/resolv.conf
# Acesso ao container
ip netns exec cnt unshare --mount --pid --fork --mount-proc=fs/merged/proc chroot fs/merged /bin/sh
```

## Documentação de comandos shell
Existe o site `explainshell.com` que podemos passar um comando shell através da seguinte URI: `https://explainshell.com/explain?cmd=<comando shell>`. Exemplo:
`https://explainshell.com/explain?cmd=iptables -I FORWARD -i meu-switch -o meu-switch -j ACCEPT`
