# DynatraceWorkshop2

## 1 - Description du projet
L'objectif de ce document est de:
1) déployer une application web multiconteneurs dans Kubernetes sur Azure et instrumenter ce cluster Kubernetes avec le OneAgentOperator
2) instrumenter avec Dynatrace une application mobile Android communiquant avec cette application

### 1.a - Présentation de l'application

L'application est une simple application test Microsoft où l'on peut voter pour son animal préféré. <br>

![Alt text](images/AzureVotingApp_webInterface.PNG?raw=true "Title")

### 1.b - Technologies utilisées

#### Docker

#### Azure

#### Kubernetes

## 2 - Pré-requis: Souscrire à un compte Azure 

Demandez à GTG l’activation de visual-studio-msdn-activation sur votre compte Dynatrace. Une fois activé, vous bénéficierez de 45€ de crédit Azure par mois, bloqués et sans besoin de mettre de carte de crédit: https://gtg.dynatrace.com/solutions/609798-visual-studio-msdn-activation

## 3 - Préparer une application pour Azure Kubernetes Service (AKS)
(Windows)

### Etape 1: Télécharger le code source de l'application

Allez à l'adresse https://github.com/Azure-Samples/azure-voting-app-redis.git et cliquer sur le bouton "Clone or Download" puis "Download ZIP" pour télécharger le projet.

### Etape 2: Tester l’application multiconteneurs dans un environnement Docker local

Dans le répertoire azure-voting-app-redis se trouvent le code source de l’application, un fichier Docker Compose précréé et un fichier manifeste Kubernetes. Vous pouvez utiliser Docker Compose pour automatiser la création d’images conteneur et le déploiement d’applications multiconteneurs.

#### 2.1 - Créer l'image conteneur et démarrer l'application
Après avoir installer Docker Compose: https://docs.docker.com/compose/install/, la commande suivante vous permet de créer l’image conteneur, téléchargez l’image Redis, puis démarrez l’application localement.
```shell
$ docker-compose up -d
```
Le fichier docker-compose.yaml définit les services de votre application afin de pouvoir les exécuter ensemble dans une environnement isolé.

#### 2.2 - Vérifier que les conteneurs ont bien été crées
Affichez les images créée:
```shell
$ docker images

REPOSITORY                   TAG        IMAGE ID            CREATED             SIZE
azure-vote-front             latest     9cc914e25834        40 seconds ago      694MB
redis                        latest     a1b99da73d05        7 days ago          106MB
tiangolo/uwsgi-nginx-flask   flask      788ca94b2313        9 months ago        694MB
```
Visualisez les conteneurs en cours d’exécution :
```shell
$ docker ps

CONTAINER ID        IMAGE             COMMAND                  CREATED             STATUS              PORTS                           NAMES
82411933e8f9        azure-vote-front  "/usr/bin/supervisord"   57 seconds ago      Up 30 seconds       443/tcp, 0.0.0.0:8080->80/tcp   azure-vote-front
b68fed4b66b6        redis             "docker-entrypoint..."   57 seconds ago      Up 30 seconds
```

#### 2.3 - Ouvrir l'application localement
Pour voir l’application en cours d’exécution, entrez http://localhost:8080 dans un navigateur web local.

#### 2.4 - Supprimer des ressources
Maintenant que la fonctionnalité de l’application a été validée, arrêtez et supprimez les instances et ressources de conteneur:
```shell
$ docker-compose down
```
:exclamation: Ne supprimez pas les images de conteneur. Lorsque l’application locale a été supprimée, vous disposez d’une image Docker qui contient l’application Azure Vote, azure-vote-front.

## 4 - Déployer et utiliser Azure Container Registry

### Etape 0
1) Connectez vous avec votre compte Dynatrace au portal Azure: https://portal.azure.com et vérifiez que vous êtes bien souscrit. <br/>
![Alt text](images/azure_subscription.PNG?raw=true "Title")

2) Téléchargez l'Azure CLI sur votre machine: https://aka.ms/installazurecliwindows
3) Ouvrez un terminal Windows et connectez vous à votre compte Dynatrace. Si la page d'authentification ne s'affiche pas automatiquement, allez sur https://aka.ms/devicelogin:
```shell
$ az login
```

### Etape 1: Création d’un Azure Container Registry

1) Créez un groupe de ressources nommé *myResourceGroup* créé dans la région *eastus*:
```shell
$ az group create --name myResourceGroup --location eastus
```
2) Créez une instance Azure Container Registry nommé *myContainerRegistryName*:
```shell
$ az acr create --resource-group myResourceGroup --name myContainerRegistryName --sku Basic
```
3) Connectez vous à ce registre (la commande devrait retourner le message *Login Succeeded*):
```shell
$ az acr login --name myContainerRegistryName
```

### Etape 2: Baliser une image conteneur et l'envoyer à une instance ACR
  Affichez la liste des images locales actuelles: 
  ```shell
  $ docker images

  REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
  azure-vote-front             latest              4675398c9172        13 minutes ago      694MB
  redis                        latest              a1b99da73d05        7 days ago          106MB
  tiangolo/uwsgi-nginx-flask   flask               788ca94b2313        9 months ago        694MB
  ```
  Obtenez l’adresse du serveur de connexion (acrLoginServer):
  ```shell
  $ az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
  ```
  Balisez votre image azure-vote-front locale avec l'acrLoginServer et le tag v1:
  ```shell
  $ docker tag azure-vote-front <acrLoginServer>/azure-vote-front:v1
  ```
  Vérifiez que le tag a été appliqué: 
  ```shell
  $ docker images
  
  REPOSITORY                                           TAG           IMAGE ID            CREATED             SIZE
  azure-vote-front                                     latest        eaf2b9c57e5e        8 minutes ago       716 MB
  mycontainerregistry.azurecr.io/azure-vote-front      v1            eaf2b9c57e5e        8 minutes ago       716 MB
  redis                                                latest        a1b99da73d05        7 days ago          106MB
  tiangolo/uwsgi-nginx-flask                           flask         788ca94b2313        8 months ago        694 MB
  ```

  Envoyez l'image à votre instance ACR:
  ```shell
  $ docker push <acrLoginServer>/azure-vote-front:v1
  ```

### Etape 3: Vérifier que l'image a bien été ajouté au registre
Liste des images qui ont été envoyées à votre instance ACR:
```shell
$ az acr repository list --name myContainerRegistryName --output table
```
```shell
Result
----------------
azure-vote-front
```

Les étiquettes d’une image spécifique:
```shell
$ az acr repository show-tags --name myContainerRegistryName --repository azure-vote-front --output table
```
```shell
Result
--------
v1
```

## 5 - Déployer un cluster Azure Kubernetes Service (AKS)

### Etape 1: Créer un principal du service
```shell
$ az ad sp create-for-rbac --skip-assignment
```
```shell
{
  "appId": "e7596ae3-6864-4cb8-94fc-20164b1588a9",
  "displayName": "azure-cli-2018-06-29-19-14-37",
  "name": "http://azure-cli-2018-06-29-19-14-37",
  "password": "52c95f25-bd1e-4314-bd31-d8112b293521",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db48"
}
```
Prenez note des valeurs de appId et de password.

### Etape 2: Configurer une authentification ACR

Obtenez l’ID de ressource ACR et mettez à jour le nom de Registre <acrName> avec celui de votre instance ACR et le groupe de ressources où se trouve cette instance:
```shell
$ az acr show --resource-group myResourceGroup --name myContainerRegistryName --query "id" --output tsv
```
Pour accorder l’accès qui permettra au cluster AKS de tirer (pull) des images stockées dans ACR, attribuez le rôle AcrPull: 
```shell
$ az role assignment create --assignee <appId> --scope <acrId> --role acrpull
```
  
  
### Etape 3: Créer un cluster Kubernetes
Créez un cluster AKS: 
```shell
$ az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --service-principal <appId> --client-secret <password> --generate-ssh-keys
```

### Etape 4: Installer l’interface de ligne de commande Kubernetes
```shell
$ az aks install-cli
```

### Etape 5: Se connecter au cluster à l’aide de kubectl
```shell
$ az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```
Vérifier la connexion à votre cluster:
```shell
$ kubectl get nodes

NAME                       STATUS   ROLES   AGE     VERSION
aks-nodepool1-28993262-0   Ready    agent   3m18s   v1.9.11
```

:exclamation: Vous pouvez également le visualiser dans le Web UI de Kubernetes en suivant les étapes de la partie #8.

## 6 - Exécuter des applications dans Azure Kubernetes Service (AKS)

### Etape 1: Mettre à jour le fichier manifeste
```shell
$ az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
```

Dans le référentiel git cloné, dans le répertoire *azure-voting-app-redis*, ouvrez le fichier manifeste *azure-vote-all-in-one-redis.yaml* avec un éditeur de texte et remplacez microsoft par le nom de votre serveur de connexion ACR:
```shell
containers:
- name: azure-vote-front
  image: microsoft/azure-vote-front:v1
```
Fichier modifié:
```shell
containers:
- name: azure-vote-front
  image: myContainerRegistryName.azurecr.io/azure-vote-front:v1
```

### Etape 2: Déployer l’application
```shell
$ kubectl apply -f azure-vote-all-in-one-redis.yaml

deployment "azure-vote-back" created
service "azure-vote-back" created
deployment "azure-vote-front" created
service "azure-vote-front" created
```

### Etape 3: Vérifier le déploiement

```shell
$ kubectl get service azure-vote-front --watch
```

Dans un premier temps, la valeur EXTERNAL-IP du service azure-vote-front est indiqué comme étant en attente (pending) :
```shell
azure-vote-front   10.0.34.242   <pending>     80:30676/TCP   7s
```
Quand l’adresse EXTERNAL-IP passe de l’état pending à une adresse IP publique réelle, utilisez CTRL-C pour arrêter le processus de surveillance kubectl.
```shell
azure-vote-front   10.0.34.242   52.179.23.131   80:30676/TCP   2m
```
Pour voir l’application en action, ouvrez un navigateur web en utilisant l’adresse IP externe de votre service

## 7 - Instrumenter le cluster Kubernetes avec le OneAgentOperator
 
### Etape 1: Installer le OneAgentOperator
Suivre les étapes d'installation ici https://www.dynatrace.com/support/help/technology-support/cloud-platforms/kubernetes/installation-and-operation/full-stack/deploy-oneagent-on-kubernetes/ 
<br/><br/>
ou ici https://github.com/Dynatrace/dynatrace-oneagent-operator
<br/><br/>
Si vous êtes confronté à des problèmes d'installation, analysez les logs: https://www.dynatrace.com/support/help/technology-support/cloud-platforms/kubernetes/installation-and-operation/full-stack/troubleshoot-oneagent-on-kubernetes/

### Etape 2: Restart le container azure-vote-front
Récupérer le nom du déploiement à restart:
```shell
$ kubectl -n dynatrace get deployments
```
Stop:
```shell
$ kubectl -n dynatrace scale deployments <name_of_deployment> --replicas=0
```
Restart:
```shell
$ kubectl -n dynatrace scale deployments <name_of_deployment> --replicas=1
```


## 8 - (Facultatif) Ouvrir le Kubernetes Web UI
Le dashboard Kubernetes peut vous aider dans le déploiement. Vous pouvez y visualiser vos nodes, pods, services et secrets de votre cluster. <br/> <br/>
Déployez le dashboard:
```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```
Ouvrez un nouveau terminal et exécutez:
```shell
$ kubectl proxy
```
Le Web UI se trouvera à l'adresse: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
<br/> <br/> 
Pour vous connectez avec un token: <br/>
- Visualisez tous les secrets du cluster avec la commande:
```shell
$ kubectl -n kube-system get secret
```
- Recherchez dans les résultats le secret avec le nom " admin-user". Copiez le token associé et utilisez le pour vous identifiez dans le Web UI.