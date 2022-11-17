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

Primeiramente, criaremos uma namespace onde ficará todos os pods, serviços, secrets e outros. Para criar, basta digitar o comanndo:
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
* No campo `kind` especificamos o que será aplicado cluster, onde nesse caso é uma Namespace e no campo `name` inserimos o nome que será dado a namespace.

Salve o arquivo como `ns.yaml` e abra um terminal no diretório onde localiza o arquivo e digite o comando:

```ruby
kubectl apply -f ns.yaml
```

Segundamente, crie um diretório onde será armazenados os arquivos yaml que são necessários para a configuração do WordPress e MySql. Dentro do diretório, crie outros dois para armazenar os arquivos separadamente das aplicações.

## Subindo o MySql

Começaremos subindo o serviço que será necessário para que o wordpress acesse o banco de dados do MySql.
Crie o arquivo mysql-service.yaml e dentro dele insira os seguintes campos:
```ruby
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
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
* É visível que, o tipo a ser criado é um Service, com o nome dado de wordpress-mysql especificando a `namespace` em que ele será inserido seguido das `labels` que define como essa aplicação será identificada por outros usuários. 
* No campo `spec`, é especificado a porta na qual o serviço do MySql usará, sendo escolhida a porta `3308` e também o `selector`, que seria um agrupamento básico primitivo no Kubernetes. 
* O campo `ClusterIP` define que, neste caso, este serviço não possuirá nenhum.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f mysql-service.yaml
```

Agora subiremos uma secret, que é onde ficará armazenado a senha para acesso ao banco de dados do MySql. Crie o arquivo mysql-password-secret.yaml e cole os campos:
```ruby
apiVersion: v1
data:
  password: <sua-senha>
kind: Secret
metadata:
  name: mysql-pass
  namespace: labwordpress
type: Opaque
```
* O tipo criado será uma secret que possui um `data` inserido nela (`password`), que será a senha que você deve inserir.

>**ATENÇÃO**: A senha inserida no secret, deverá estar criptografada em base64. Para criptografar, basta digitar no terminal o comando `echo -n 'sua-senha' | base64` que irá imprimir a mensagem criptografada e inserir no campo password no arquivo do secret.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f mysql-password-secret.yaml
```

Para solitação de armazenamento dos dados, iremos implementar um PersistentVolumeClaim, que solicitará o acesso ao PersistentVolume, que será criado automaticamente pelo Docker Desktop, usando um StorageClass padrão onde terá os dados armazenados das aplicações em questão. Isso é utilizado caso algum pod morra e não aconteça a perda dos dados já salvos.

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
* Possui também o `accessModes` que define os modos de acesso do PersistentVolume que nesse caso será de Leitura e Escrita por um nó único. 
* Em seguida, há o `storage`, que é o tamanho que vai ser utilizado para armazenar os dados.

Agora aplicaremos as configurações do PersistentVolumeClaim, digitando o comando no terminal onde o arquivo se encontra:

```ruby
kubectl apply -f mysql-password-secret.yaml
```

Para finalizar o Deploy do MySql, faremos o Deployment do container que irá baixar a imagem e subir a aplicação. Para isso, criaremos o arquivo mysql-deployment.yaml e inserir os seguintes campos:
```ruby
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
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
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
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
* O Deployment que irá fazer o download da imagem do mysql na versão 5.6 que é especificado na parte de `spec` do arquivo, onde possui também uma variavél de ambiente, sendo ela a senha do ROOT do MySql que vai ser substítuida pelo Secret(`mysql-pass`) configurado anteriormente. Também possui a porta do container 3308 definida. 

* Há também, o campo `strategy` definido como Recreate que caso aconteça algum erro ao subir o Pod, ele ficará tentando até que erro seja corrigido e não aconteça mais.

* No campo `volumeMounts` é usado para montar o PersistentVolume no diretório /var/lib/mysql dentro do container, e em `volumes`, ele solicita a autorização ao PersistentVolume através do PersistentVolumeClaim já criado anterirmente.

Para finalizar, basta inserir o comando para aplicar o deploy:
```ruby
kubectl apply -f mysql-deployment.yaml
```
E por fim a aplicação do MySql já estará rodando.

## Subindo o Wordpress

Com os arquivos de configuração do Mysql devidamente configurados, podemos configurar os arquivos do Wordpress começando pelo seu serviço.

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
* Observando o campo `kind`, vemos que o tipo a ser criado é um Service, com o nome dado de wordpress especificando a `namespace` em que ele será inserido seguido das `labels` que define como essa aplicação será identificada por outros usuários. 
* No campo `spec`, é especificado a porta na qual o serviço do wordpress usará, sendo escolhida a porta `80`.
* Por fim, é definido o tipo de serviço que será usado para a conexão, sendo ele `ClusterIP`.

Para aplicar as confirações, digite o comando no terminal onde o arquivo se encontra:
```ruby
kubectl apply -f wordpress-service.yaml
```

Para que tenha o acesso ao serviço do Wordpress de para quem está fora do cluster, é necessário a criação de um Ingress. Para que ele funcione, o mesmo necessita de um controlador, pois sem ele o Ingress não tem efeito. O controlador a ser usado será o nginx que é simples de se instalar e configurar.

Para instalar o nginx, basta dar um apply no seguinte comando:
```ruby
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```
Este comando irá configurar o controlador em uma namespace a parte com todas configurações já definidas(ClusterRole, Role, Jobs, Pods, etc.) que são necessários para o funcionamento do mesmo. Depois de um tempo, todos os pods devem estar funcionando. Digite o comando a seguir para conferir o funcionamento dos pods:

```ruby
kubectl get pods -n ingress-nginx
```

Em seguida, com o nginx em funcionamento, aplicaremos o ingress ao namespace para o acesso ao serviço. Para isso basta criar um arquivo com o nome wordpress-ingress.yaml e inserir os seguintes campos:
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

Para solitação de armazenamento dos dados do WordPress, o PersistentVolumeClaim. Para esta implementação, basta criar um arquivo `yaml`, cujo o nome dado será `wordpress-pvc.yaml` e inserir os seguintes campos:

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
* Em seguida, há o `storage`, cujo ele também possui um tamanho de 3Gi para armazenar os dados.

Agora aplicaremos as configurações do PersistentVolumeClaim, digitando o comando no terminal onde o arquivo se encontra:

```ruby
kubectl apply -f wordpress-pvc.yaml
```

Para finalizar, faremos o Deployment do container que irá baixar a imagem e subir a aplicação. Para isso, criaremos o arquivo `wordpress-deployment.yaml` e inserir os seguintes campos:
```ruby
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: labwordpress
  labels:
    app: wordpresss
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
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
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

* O Deployment irá fazer o download da imagem do wordpress na versão 4.8-apache, onde possui duas variaveis de ambiente, sendo uma, a `WORDPRESS_DB_HOST` que aponta para o serviço do mysql para se conectar a database, e outra, MYSQL_DB_PASSWORD sendo a senha do ROOT do MySql para conectar a database que vai ser substítuida pelo Secret(`mysql-pass`) configurado anteriormente. Também possui a porta do container 80 definida.

* Há também, o campo `strategy` definido como Recreate que caso aconteça algum erro ao subir o Pod

* No campo `volumeMounts` é usado para montar o PersistentVolume no diretório /var/www/html dentro do container, e em `volumes`, ele solicita a autorização ao PersistentVolume através do PersistentVolumeClaim já criado anterirmente.

Para finalizar, basta inserir o comando para aplicar o deploy:
```ruby
kubectl apply -f wordpress-deployment.yaml
```
E por fim a aplicação do Wordpress já estará rodando.

# Testando a aplicação

Para acessar a página da aplicação do WordPress, insira o DNS que foi incluído no Ingress criado do wordpress. Para obter este link, digite o comando no terminal:
`kubectl get ing wordpress -n labwordpress`
O link a incluir no navegador será o que se encontra na aba HOSTS

Ao acessar a página, você será redirecionado para a página inicial de configuração do WordPress, podendo assim configurar da forma em que você deseja escolhendo o idioma de preferência.

![img3](https://user-images.githubusercontent.com/108817932/202470571-4f2e2b80-e402-4e8e-ab46-36f87ab54564.png)

Em seguida, você deve preencher as informações pedidas para finalizar a instalação do WordPress.

![img4](https://user-images.githubusercontent.com/108817932/202471270-4e5b39ff-da10-40e7-bb2e-7c140ab7f43f.png)

E finalmente ter acesso ao dashboard da aplicação.