# Lab04 - Criação de pods K8S
> [Voltar](../README.md)

Nesse laboratório, iremos criar pods Kubernetes, utilizando o [k3s](https://k3s.io/) da CNCF para criação do cluster K8S, através de uma VM Ubuntu Server 20.04. Utilizar um ambiente [Ubuntu 20.04 no Killercoda](https://killercoda.com/playgrounds/scenario/ubuntu). Utilizaremos o programa de linha de comando `kubectl` para administração do cluster.

## 1. Preparação do ambiente para laboratório
Para esse laboratório, precisaremos realizar a instalação do Microk8s no host, que chamaremos de `Node 1`.
```shell
# Instalar K3S com kubectl
curl -sfL https://get.k3s.io | sh -
```

Verificar se o cluster está funcionando:
```shell
# Obter informações dos nós do cluster K8S
kubectl get nodes
# Observe que possuímos uma única instância de K8S
```

## 2. Kubernetes Hello-World
### No Node 1
Acessar a pasta home do usuário e criar a pasta lab04:
```shell
cd
mkdir lab04
cd lab04
```

---
:exclamation::exclamation::exclamation: `Atenção`: cada espaço no arquivo de configuração faz diferença, dado que os arquivos de configuração serão criados utilizando o formato YAML. Criar os arquivos de configuração exatamente como mostrados nesse laboratório. Utilize ferramentas online de validação do YAML em caso de dúvidas, como exemplo o [YAML Checker](https://yamlchecker.com/).

Após alterar cada arquivo de configuração, para salvar, seguir a seguinte sequência de comandos:
- Pressione Ctrl+X
- Pressione Y
- Pressione Enter

---

Criar o arquivo de configuração de volume para o nosso serviço:
```shell
nano pvc.yaml
```

Incluir o seguinte conteúdo:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim # Permite criarmos volumes persistentes para nossos containers
metadata:
  name: hello-world-pvc # Nome da configuração de PVC
  labels:
    app: hello-world # Label aplicada no PVC criado, contendo o nome da aplicação
spec:
  accessModes:
    - ReadWriteOnce # Permite que único container acesse o mesmo volume persistente
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi # Limita o armazenamento em disco do volume em 1GB
```


Criar o arquivo de configuração de deployment para criação dos pods do nosso serviço:
```shell
nano deployment.yaml
```

Incluir o seguinte conteúdo:
```yaml
apiVersion: apps/v1
kind: Deployment # Permite especificarmos nossos containers para execução do serviço Hello-World
metadata:
  name: hello-world-deployment # Nome da configuração de Deployment
  labels:
    app: hello-world # Label aplicada no Deployment criado, contendo o nome da aplicação
spec:
  replicas: 3 # Quantidade de réplicas do nosso serviço Hello-World
  selector:
    matchLabels:
      app: hello-world # Indicamos para aplicar a configuração de réplicas para containers que possuem a label `app: hello-world`
  template:
    metadata:
      labels:
        app: hello-world # Label aplicada nos pods criados, contendo o nome da aplicação, utilizado pelo matchLabel acima
    spec:
      containers:
        - name: hello-world # Nome do container
          image: containertools/hello-world # Imagem do container
          imagePullPolicy: "Always" # Indica que a imagem do container será sempre atualizada
          ports:
            - containerPort: 8080 # Porta do container
              name: http-port # Nome da porta, para uso no arquivo de configuração do tipo Service
          env: # Variáveis de ambiente a serem inseridas no container
            - name: FILES_BASEPATH
              value: /vol
          volumeMounts: # Configuração de volumes para o container
            - mountPath: /vol # Caminho do volume no container
              name: hello-world-vol # Nome do volume a ser utilizado (conforme lista de volumes a seguir)
      volumes: # Lista de volumes a serem utilizados em containers declarados acima
        - name: hello-world-vol # Nome do volume
          persistentVolumeClaim:
            claimName: hello-world-pvc # Nome da configuração de PVC para esse volume
      securityContext:
        fsGroup: 0 # Muda o owner dos volumes montados em containers, para o usuário root
        runAsUser: 0 # Executa o container utilizando o usuário root
        # Obs.: Por questões de segurança, evitar utilizar o usuário root em ambiente produtivo
```

Criar o arquivo de configuração de service para acessarmos os pods do nosso serviço:
```shell
nano service.yaml
```
Incluir o seguinte conteúdo:
```yaml
apiVersion: v1
kind: Service # Permite criarmos um serviço para acessarmos os pods criados pelo deployment, a partir de um único IP virtual
metadata:
  name: hello-world-service # Nome da configuração de Service
  labels:
    app: hello-world # Label aplicada no Service criado, contendo o nome da aplicação
spec:
  selector:
    app: hello-world # Indicamos para distribuir requisições do service para pods que possuem a label `app: hello-world`
  ports:
  - name: http-sv-port # Nome da porta que iremos expor no service
    port: 80 # Porta que iremos expor no service
    targetPort: http-port # Porta dos pods que iremos distrubuir as requisições
```
Nesse exemplo, o service criado é do tipo `ClusterIP`, sendo acessível apenas de dentro do cluster K8S. Criaremos um `Ingress` para permitir o acesso ao nosso serviço de maneira externa ao cluster.

Criar o arquivo de configuração de ingress para permitir acessarmos nosso serviço a partir da URL gerada da página de acesso na porta 80 em `Traffic / Ports`:
```shell
nano ingress.yaml
```
Incluir o seguinte conteúdo:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress # Permite criarmos um ingress para acessarmos nosso serviço de maneira externa ao cluster K8S
metadata:
  name: hello-world-ingress # Nome da configuração de Ingress
spec:
  rules:
  - http:
      paths:
      - path: / # Permite acesso a todos os recursos/páginas do nosso serviço (/qualquer/caminho/possivel)
        pathType: Prefix # Indica que o path indicado acima, trata-se de um prefixo da URL requisitada
        backend:
          service:
            name: hello-world-service # Nome do service que será direcionada a requisição
            port:
              number: 80 # Porta do service que será direcionada a requisição
```

Aplicar as configurações criadas:
```shell
# Aplicar a configuração de PVC (hello-world-pvc)
kubectl apply -f pvc.yaml
# Aplicar a configuração de Deployment (hello-world-deployment)
kubectl apply -f deployment.yaml
# Aplicar a configuração de Service (hello-world-service)
kubectl apply -f service.yaml
# Aplicar a configuração de Ingress (hello-world-ingress)
kubectl apply -f ingress.yaml
# Para aplicar todos os arquivos .yaml da pasta corrente, utilizar o seginte comando:
kubectl apply -f .
```

Para listar as configurações aplicadas:
```shell
# Listar os recursos PVC criados
kubectl get pvc
# Listar os recursos PV criados
kubectl get pv
# Listar os recursos Deployment criados
kubectl get deployment
# Listar os recursos Service criados
kubectl get service
# Listar os recursos Ingress criados
kubectl get ingress
# Listar os pods criados
kubectl get pods
# Para maiores detalhes em cada comando `kubectl get` executado acima, adicionar o parâmetro `-o wide`. Exemplo:
kubectl get deployment -o wide
# Para acompanhar a subida das 3 réplicas de pod especificadas no deployment, adicionar o parâmetro `-w`
kubectl get deployment -o wide -w
# Quando a coluna `Ready` mostrar `3/3` para o deployment hello-world-deployment, pressionar Ctrl+C para interromper o comando
# Caso ocorra algum erro ao tentar criar um recurso, podemos utilizar o comando `kubectl describe` para verificar eventuais eventos de erro gerados para o recurso:
kubectl describe pods
# Podemos realizar o describe de um pod específico (trocar para o nome do pod que desejar):
kubectl describe pods hello-world-deployment-7c56c6f587-bl2xf
```

Caso precise alterar uma configuração, editar o arquivo de configuração correspondente e executar o comando `kubectl apply -f nome_do_arquivo.yaml` novamente.

Liste os pods criados:
```shell	
kubectl get pods
```
```
NAME                                      READY   STATUS    RESTARTS   AGE
hello-world-deployment-7c56c6f587-6l5cp   1/1     Running   0          4m22s
hello-world-deployment-7c56c6f587-brrcc   1/1     Running   0          4m22s
hello-world-deployment-7c56c6f587-kldj8   1/1     Running   0          4m22s
```

Escolha um pod para entrarmos em seu container. Exemplo:
```shell
kubectl exec -it hello-world-deployment-7c56c6f587-6l5cp -- /bin/sh
```

### No container
```shell
cd /vol
echo "Arquivo de teste de dentro do container" > no-container.txt
exit
```

Em `Traffic Port Accessor`, acessar porta 80 e verificar se o arquivo criado é listado na página.

### No Node 1
Vamos obter os detalhes do volume criado através da configuração PVC.
```shell
# Obter os volumes persistentes criados
kubectl get pv
```
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS        REASON   AGE
pvc-1925a2db-77e7-4b27-aae6-961dfd455d25   1Gi        RWX            Delete           Bound    default/hello-world-pvc   microk8s-hostpath            11m
```
```shell
# Obter o detalhe do volume persistente criado
kubectl describe pv pvc-1925a2db-77e7-4b27-aae6-961dfd455d25
```
```
Name:              pvc-46c8aa5e-656b-45e7-8e2e-87ec56240629
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: rancher.io/local-path
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-path
Status:            Bound
Claim:             default/hello-world-pvc
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          1Gi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [ubuntu]
Message:           
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/rancher/k3s/storage/pvc-46c8aa5e-656b-45e7-8e2e-87ec56240629_default_hello-world-pvc
    HostPathType:  DirectoryOrCreate
Events:            <none>
```
Observer que o StorageClass utilizado por esse volume persistente é do tipo `local-path`, que foi instalado junto ao k3s. Essa classe de storage realiza a montagem do volume local do host (`/var/lib/rancher/k3s/storage/<namespace-nome_da_config_pvc-nome_do_pvc>`) dentro do caminho especificado no container. 

Obter o caminho do volume em `Path:`. Exemplo: ` /var/lib/rancher/k3s/storage/pvc-46c8aa5e-656b-45e7-8e2e-87ec56240629_default_hello-world-pvc`

```shell
# Listar os arquivos do volume a partir do host
sudo ls -la /var/lib/rancher/k3s/storage/pvc-46c8aa5e-656b-45e7-8e2e-87ec56240629_default_hello-world-pvc
# Adicionar um arquivo no volume a partir do host
sudo echo "Olá do host" > /var/lib/rancher/k3s/storage/pvc-46c8aa5e-656b-45e7-8e2e-87ec56240629_default_hello-world-pvc/no-host.txt
```

Atualizar a página de acesso na porta 80 e verificar se o arquivo criado é listado na página.

Obtenha a relação de pods criados:
```shell
kubectl get pods -o wide
```
```
NAME                                      READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
hello-world-deployment-5667c58f78-q89s5   1/1     Running   0          11m   10.42.0.12   ubuntu   <none>           <none>
hello-world-deployment-5667c58f78-cbzgn   1/1     Running   0          11m   10.42.0.10   ubuntu   <none>           <none>
hello-world-deployment-5667c58f78-d7g4s   1/1     Running   0          11m   10.42.0.11   ubuntu   <none>           <none>
```

Fique realizando o refresh da página de acesso na porta 80 e observe que o IP e Hostname variam a cada refresh. O Kubernetes distribuí proporcionalmente as requisições para os pods que correspondem ao service `hello-world-service` que criamos.

Escolha um pod para deletarmos:
```shell
kubectl delete pod hello-world-deployment-7c56c6f587-6l5cp
```

Fique realizando o refresh da página de acesso na porta 80 e observe que um novo pod foi criado automaticamente, para garantir que o número de réplicas seja 3, conforme configurado no `hello-world-deployment`.

Verificar o pod novo criado automaticamente pelo Kubernetes:
```shell
kubectl get pods -o wide
```

Para destruir todos os recursos criados, executar o seguinte comando:
```shell
# Deletar todos os recursos listados nos arquivos .yaml da pasta corrente
kubectl delete -f .
```
