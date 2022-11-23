# Implantando WordPress + MySql com o Kubernetes

# Pré-requisitos
Possuir o Docker e o Kubernetes instalados na máquina

# Instalando o Docker e Kubernetes
Para utilizar do Docker e Kubernetes, usaremos o Docker Desktop. Ele é a maneira mais fácil de executar o Kubernetes em sua máquina local, ele fornece um cluster Kubernetes totalmente certificado e gerencia todos os componentes para você.

## 1. Instalando o Docker Desktop

Para baixar o Docker Desktop, clique no link a seguir e realize a instalação: [Download Docker Desktop](https://www.docker.com/products/docker-desktop/)

## 2. Habilitando o Kubernetes
Após realizar a instalação do Docker Desktop, é necessário habilitar o Kubernetes. Para isso basta ir nas configurações do Docker Desktop clicando na engrenagem.

![img1](https://user-images.githubusercontent.com/78823528/202195146-99156dab-7833-4446-94ef-affc47c48408.png)

Em seguida, vá em Kubernetes e clique em Enable Kubernetes, que iniciará um cluster kubernetes já configurado para o uso. O processo pode demorar em torno de 5 minutos dependendo dá máquina que está sendo utilizada.

![img2](https://user-images.githubusercontent.com/78823528/202195991-2e4609b8-415e-46d3-ae73-581c597d50ae.png)

Para conferir se está tudo funcionando, digite `kubectl get nodes` que deve retornar o nó principal criado pelo Docker Desktop.

# Preparando o ambiente
Neste ambiente usaremos arquivos no formato `yaml` para aplicar as devidas configurações das aplicações que serão implementadas.

Primeiramente, crie um diretório onde será armazenados os arquivos yaml que são necessários para a configuração do WordPress e MySql. Em seguida criaremos uma namespace onde ficará todos os pods, services, secrets e outros. Para criar, basta digitar o comando:
```ruby
kubectl create namespace <nome-da-namespace>
```
ou criar um arquivo de configuração usando um editor de texto de sua preferência. Dentro deste arquivo insira os seguintes campos:

```ruby
apiVersion: v1
kind: Namespace
metadata:
  name: labwordpress
```
* No campo `kind` especificamos o que será aplicado cluster, onde nesse caso é uma Namespace e no campo `metadata`, são os metadados que serão usados para especificar algumas configurações como o campo `name` onde inserimos o nome que será dado a namespace, que neste caso é `labwordpress`.

Salve o arquivo como `ns.yaml` e abra um terminal no diretório onde se localiza o arquivo e digite o comando:

```ruby
kubectl apply -f ns.yaml
```

Ainda no diretório crie outros dois diretórios para armazenar os arquivos separadamente das aplicações.

## Subindo o MySql

Começaremos subindo o serviço que será necessário para que o WordPress acesse o banco de dados do MySql.
Crie o arquivo mysql-service.yaml e dentro dele insira os seguintes campos:
```ruby
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: labwordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 3308
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```
* É visível que, o tipo a ser criado é um Service, com o nome dado de mysql especificando a `namespace` em que ele será inserido seguido das `labels` que define como essa aplicação será identificada por outros usuários. 
* No campo `spec`, é especificado a porta na qual o serviço do MySql usará, sendo escolhida a porta `3308` e também o `selector`, que seria um agrupamento básico primitivo no Kubernetes para identificação para outros Pods.
* O campo `ClusterIP` define o IP de acesso ao serviço em questão que, neste caso, este serviço não possuirá nenhum, pois esse service é de conexão somente interna que será feita pelo WordPress.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f mysql-service.yaml
```

Subiremos em seguida, o ConfigMap, que é usado para armazenar dados não-confidenciais em pares chave-valor. Ele será usado no Deployment para aplicar a o usuário, nome da database e  que serão utilizadas.

Crie o arquivo mysql-configmap.yaml e insira nele os seguintes campos:
```ruby
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  namespace: labwordpress
data:
  db: wordpressdb
  user: wordpress
  mysql-service: mysql
```
* O `kind` desse arquivo é um ConfigMap onde possui um `data` que é seguido das chave-valores que serão usadas no arquivo de deployment para inserir as variaveis.
* O `db` define o nome da database usada, `user` o usuário, e `mysql-service` o serviço que o WordPress fará a conexão para se conectar ao MySql.

Para aplicar está implementação, basta digitar no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f mysql-configmap.yaml
```

Agora subiremos uma secret, que é onde ficará armazenado a senha para acesso ao banco de dados do MySql. Crie o arquivo mysql-password-secret.yaml e cole os campos:
```ruby
apiVersion: v1
data:
  rootpass: <senha-do-root>
  userpass: <senha-do-usuario-wordpress>
kind: Secret
metadata:
  name: mysql-pass
  namespace: labwordpress
type: Opaque
```
* O tipo criado será uma secret que possui `data` inserido nela , que serão as senhas para o usuario root e do wordpress, respectivamente.

>**ATENÇÃO**: A senha inserida no secret, deverá estar criptografada em base64. Para criptografar, basta digitar no terminal o comando `echo -n 'sua-senha' | base64` que irá imprimir a mensagem criptografada e inserir no campo password no arquivo do secret.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f mysql-password-secret.yaml
```

Para o armazenamento dos dados, iremos implementar um PersistentVolumeClaim, que irá requerir ao PersistentVolume, que será criado automaticamente por uma StorageClass padrão, a `hostpath`(já criada pelo Docker Desktop), ao aplicar está configuração, onde terá os dados armazenados das aplicações em questão no disco no nó do cluster. Isso é utilizado caso os pods morram, evita que os dados já salvos não sejam perdidos e ao subir novamente os pods, os dados sejam recuperados.

Para esta implementação, basta criar um arquivo `yaml`, cujo o nome dado será `mysql-pvc.yaml` e inserir os seguintes campos:
```ruby
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: labwordpress
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
* O tipo definido é de PersistentVolumeClaim, com o nome de `mysql-pv-claim` inserido na namespace `labwordpress` com o label já utilizado anteriormente, `app: wordpress`. 
* Possui também o `accessModes` que define os modos de acesso do PersistentVolume que nesse caso será de Leitura e Escrita. 
* Em seguida, há o `storage`, que é o tamanho que vai ser utilizado para armazenar os dados.

Agora aplicaremos as configurações do PersistentVolumeClaim, digitando o comando no terminal onde o arquivo se encontra:

```ruby
kubectl apply -f mysql-pvc.yaml
```

Para finalizar o Deploy do MySql, faremos o Deployment do container que irá baixar a imagem e subir a aplicação. Para isso, criaremos o arquivo mysql-deployment.yaml e inserir os seguintes campos:
```ruby
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: labwordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0.31
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: rootpass
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mysql-configmap
              key: db
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: mysql-configmap
              key: user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: userpass
        ports:
        - containerPort: 3308
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage-lab
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage-lab
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```
* O Deployment que irá criar um container com a imagem do mysql na versão 8.0.31 que é especificado na parte de `spec` do arquivo. Esse container ficará dentro de um pod que será criado na namespace inserida 

* Possui também variaveis de ambiente, que apontam para o Secret(`secretKeyRef`) e para o ConfigMap(`configMapKeyRef`) já configurados anteriormente, apresentando seus respectivos nomes e valores.

* No campo `ports`, localiza a porta do container, escolhida como 3308. 

* Há também, o campo `strategy` definido como Recreate que caso aconteça algum erro ao subir o Pod, ele ficará tentando até que erro seja corrigido e não aconteça mais.

* No campo `volumeMounts` é usado para montar o volume no diretório /var/lib/mysql dentro do container, e em `volumes`, ele solicita ao PersistentVolume através do PersistentVolumeClaim já criado anteriormente para armazenamento dos dados do MySql na máquina host.

Para finalizar, basta inserir o comando para aplicar o deploy:
```ruby
kubectl apply -f mysql-deployment.yaml
```
E por fim a aplicação do MySql já estará rodando..

## Subindo o WordPress

Com os arquivos de configuração do Mysql devidamente configurados, podemos configurar os arquivos do WordPress começando pelo seu serviço.

Crie o arquivo wordpress-service.yaml e dentro dele insira os seguintes campos:

```ruby
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wordpress
  name: wordpress
  namespace: labwordpress
spec:
  ports:
  - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: ClusterIP
```
* Observando o campo `kind`, vemos que o tipo a ser criado é um Service, com o nome dado de `wordpress` especificando a `namespace` em que ele será inserido seguido das `labels` que define como essa aplicação será identificada por outros usuários e pods.
* No campo `spec`, é especificado a porta na qual o serviço do wordpress usará, sendo escolhida a porta `80`.
* Por fim, é definido o tipo de serviço que será usado para a conexão, sendo ele `ClusterIP`, que irá gerar um IP.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f wordpress-service.yaml
```

Para o armazenamento dos dados do WordPress, cria-se o PersistentVolumeClaim. Para esta implementação, basta criar um arquivo `yaml`, cujo o nome dado será `wordpress-pvc.yaml` e inserir os seguintes campos:

```ruby
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  namespace: labwordpress
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
* O tipo definido é de PersistentVolumeClaim, com o nome de `wp-pv-claim` inserido na namespace `labwordpress` com o label já utilizado anteriormente, `app: wordpress`. 
* Em seguida, há o `storage`, que define um tamanho de 3Gi para armazenar os dados.

Agora aplicaremos as configurações do PersistentVolumeClaim, digitando o comando no terminal onde o arquivo se encontra:

```ruby
kubectl apply -f wordpress-pvc.yaml
```

Em seguida, faremos o Deployment do container que irá baixar a imagem e subir a aplicação. Para isso, criaremos o arquivo `wordpress-deployment.yaml` e inserir os seguintes campos:
```ruby
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: labwordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:6.1.1
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: mysql-configmap
              key: mysql-service
        - name: WORDPRESS_DB_USER
          valueFrom:
            configMapKeyRef:
              name: mysql-configmap
              key: user
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: userpass
        - name: WORDPRESS_DB_NAME
          valueFrom:
            configMapKeyRef:
              name: mysql-configmap
              key: db
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage-lab
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage-lab
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

* O Deployment irá criar um container com a imagem do wordpress na versão 6.1.1 que ficará inserido em pod automaticamente criado pelo Deployment, onde possui quatro variaveis de ambiente, onde a `WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER` e `WORDPRES_DB_NAME` apontam para ConfigMap feito anteriormente. Já a variavel `WORDPRESS_DB_PASSWORD` aponta para o secret.

* No campo `port` possui a porta do container definida por 80.

* Há também, o campo `strategy` definido como Recreate que caso aconteça algum erro ao subir o Pod.

* No campo `volumeMounts` é usado para montar o volume no diretório /var/www/html dentro do container, e em `volumes`, ele solicita o PersistentVolume através do PersistentVolumeClaim já criado anteriormente para armazenamento dos dados do WordPress na máquina host.

Para finalizar, basta inserir o comando para aplicar o deploy:
```ruby
kubectl apply -f wordpress-deployment.yaml
```

Para que tenha o acesso ao serviço do Wordpress fora do cluster, é necessário a criação de um Ingress. Para que ele funcione, o mesmo necessita de um controlador, pois sem ele o Ingress não tem efeito. O controlador a ser usado será o nginx que é simples de se instalar e configurar.

Para instalar o nginx, basta inserir o seguinte comando:
```ruby
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```
Este comando irá configurar o controlador em uma namespace a parte com todas configurações já definidas(ClusterRole, Role, Jobs, Pods, etc.) que são necessários para o funcionamento do mesmo. Depois de um tempo, todos os pods devem estar funcionando.
Digite o comando a seguir para conferir o funcionamento dos pods:
```ruby
kubectl get pods -n ingress-nginx
```

Para finalizar, com o nginx em funcionamento, aplicaremos o ingress ao namespace para o acesso ao serviço. Para isso basta criar um arquivo com o nome wordpress-ingress.yaml e inserir os seguintes campos:
```ruby
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress
  namespace: labwordpress
spec:
  ingressClassName: nginx
  rules:
  - host: lab-wordpress.com
    http:
      paths:
      - backend:
          service:
            name: wordpress
            port:
              number: 80
        path: /
        pathType: Prefix
```
* O kind definido é o `Ingress`, sendo aplicado na namespace do labwordpress onde se encontra o serviço do mesmo em que ele irá se comunicar.
* No `spec`, onde possui o ingressClassName, ele aponta para o nginx que foi criado pelo nginx. Também possui um host, onde é colocado o DNS (`lab-wordpress.com`) que aponta para o ip do WordPress(neste caso, localhost, configurado no arquivo hosts da máquina).
* Ainda no dentro do `spec`, possui um `backend` que define o serviço em que o ingress irá redirecionar e a porta escolhida.
* Por fim, o `path` que aponta o caminho após o DNS que será seguido com seu tipo. 
Para aplicar esta configuração, basta digitar o comando no terminal:

```ruby
kubectl apply -f wordpress-ingress.yaml
```
E por fim a aplicação do Wordpress já estará rodando.

# Testando a aplicação

Antes de realizar o teste, confira se está tudo rodando de forma correta. Para isso, basta digitar o comando:
```ruby
kubectl get all -n wordpress
```
Este comando deve retornar algumas informações sobre os serviços, deployments e pods já configurados.

Para acessar a página da aplicação do WordPress, insira o DNS que foi incluído no Ingress criado do wordpress. Para obter este link, digite o comando no terminal:
`kubectl get ing wordpress -n labwordpress`
O link a incluir no navegador será o que se encontra na aba HOSTS

Ao acessar a página, você será redirecionado para a página inicial de configuração do WordPress, podendo assim configurar da forma em que você deseja escolhendo o idioma de preferência.

![img3](https://user-images.githubusercontent.com/108817932/202470571-4f2e2b80-e402-4e8e-ab46-36f87ab54564.png)

Em seguida, você deve preencher as informações pedidas para finalizar a instalação do WordPress.

![img4](https://user-images.githubusercontent.com/108817932/202471270-4e5b39ff-da10-40e7-bb2e-7c140ab7f43f.png)

E assim ter acesso ao dashboard da aplicação.
