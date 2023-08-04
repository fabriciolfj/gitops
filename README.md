# gitops

## Criando imagem em app java
- jib não precisa de um container runtime para gerar a imagem
```
mvn compile com.google.cloud.tools:jib-maven-plugin:3.2.0:build -Dimage=quay.io/fabricio211/jib-example:latest -Djib.to.auth.username=<username> -Djib.to.auth.password=<password>

```

## Alternativa docker
- temos o buildah
- abaixo um comando para gerar uma imagem 
```
buildah bud -f Dockerfile -t quay.io/fabricio211/test
```

## OLM
- operator lifecycle manager, instalar/desinstalar software no kubernetes

## Shipwright
- para criar imagens de contêiner no kuberentes
```
kubectl apply -f \
  https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.30.0/release.yaml

kubectl apply -f \
  https://github.com/shipwright-io/build/releases/download/v0.7.0/release.yaml

kubectl apply -f \
  https://github.com/shipwright-io/build/releases/download/v0.7.0/sample-strategies.yaml
```
- através dele podemos criar imagens, enviar a um registry e executar as mesmas
```
kubectl get builds
//criar umagem
kubectl create -f arquivo.yml
```

- caso queira impimir o resultado de um apply, sem aplicar no cluster
```
kubectl apply --dry-run=client -o yaml \ 
              -k ./ 
```

## kustomize
- para aplicar um recuso utilizando kustomize
```
kustomize build . | kubectl apply -f -
```
- obs: precisamos do arquivo kustomization.yaml configurado e o comando executado na raiz (aonde encontra-se o arquivo)
- podemos atualizar a imagem da app no arquivo kustomize
```
kustomize edit set image lordofthejars/pacman-kikd:1.0.2
```
- usando o kustomize para o configmap, ajuda a gerar e a recriar os pods, uma vez que ele também atualiza o nome do configmap no deployment
- obs: quando alteramos apenas o configmap, os podemos não são recriados automatizamente.
- podemos referenciar um arquivo como nosso configmap
  - arquivo de nome connection.properties
```
db-url=prod:4321/db
db-username=ada
```
  - conteudo do yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
namespace: prod
images:
  - name: lordofthejars/pacman-kikd
    newTag: 1.1.0
configMapGenerator:
  - name: pacman-configmap
    files:
      - ./connection.properties
```
  - o configmap gerado, tera como chave o nome do arquivo, conforme demonstrado abaixo
```
apiVersion: v1
data:
  connection.properties: |-
    db-url=prod:4321/db
    db-username=ada
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"connection.properties":"db-url=prod:4321/db\ndb-username=ada"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"pacman-configmap-g9dm2gtt77","namespace":"prod"}}
  creationTimestamp: "2023-07-14T01:58:07Z"
  name: pacman-configmap-g9dm2gtt77
  namespace: prod
  resourceVersion: "713285"
  uid: 2707fa44-d5ec-4497-b415-05214c707b19

```

## helm
- um gerenciador de pacotes para kubernetes
- podemos criar um template e colocar valores nele em tempo de execução, através de um Chart.yaml ou values.yaml
- comando para buildar
```
helm template .
```
- comando para instalar (execute na raiz)
```
helm install pacman .
```
- para empacotar afim de distribuir seu Chart
```
helm package .
```

#### uso de repositorios remotos
- podemos instalar charts de repositórios remotos, para isso precisamos primeiro adicionar:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
- atualizar
```
helm update .
```
- por fim instalar
```
helm install my-db --set postgresql.postgresqlUsername=my-default,postgresql.postgresqlPassword=postgres,postgresql.postgresqlDatabase=mydb,postgresql.persistence.enabled=false bitnami/postgresql
```
- caso seu chart tenha alguma dependencia (verificar diretorio music)
```
helm dependency update .
```
- para instalar um chart criado
```
helm install nome-test .
```

## tekton
- um sistem ci/cd native em nuvem sobre o kubernetes
- é instalado dentro do k8s
- instalacao:
```
kubectl apply \
-f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.38.0/release.yaml

kubectl apply \
-f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.20.1/release.yaml

kubectl apply \
-f https://storage.googleapis.com/tekton-releases/dashboard/previous/v0.28.0/tekton-dashboard-release.yaml
```
- para acessar o dashboard kubectl port-forward svc/tekton-dashboard 9097:9097 -n tekton-pipelines
- para listar as tasks que nos criamos:
```
kubectl get tasks
```

### repositorios privados
- para uso de repositorios privados, apos criar o secret, podemos anotar o mesmo no tekton: kubectl annotate secret git-secret "tekton.dev/git-0=https://github.com"
- apos anotar, anexar a um service account
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-bot-sa
secrets:
  - name: git-secret 1
```
- abaixo um exemplo via cli, vinculando o service account para realizar o CD, com base no app no repositorio privado
```
tkn task start build-app \
  --serviceaccount='tekton-bot-sa' \ 
  --param url='https://github.com/gitops-cookbook/tekton-greeter-private.git' \ 
  --param contextDir='quarkus' \
  --workspace name=source,emptyDir="" \
  --showlog
```
- para enviar a um registry, devemos:
```
criar um secret
kubectl create secret docker-registry container-registry-secret \
  --docker-server='quay.io' \
  --docker-username='fabricio211' \
  --docker-password='megatron12'
  
criar um serviceaccount
kubectl create serviceaccount tekton-registry-sa

vincular o secret
kubectl patch serviceaccount tekton-registry-sa \
  -p '{"secrets": [{"name": "container-registry-secret"}]}'
  
por fim configurar a task 
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: build-app-run-
  labels:
    app.kubernetes.io/managed-by: tekton-pipelines
    tekton.dev/task: build-app
spec:
  params:
    - name: contextDir
      value: quarkus
    - name: revision
      value: master
    - name: sslVerify
      value: "false"
    - name: subdirectory
      value: ""
    - name: tlsVerify
      value: "false"
    - name: url
      value: https://github.com/gitops-cookbook/tekton-tutorial-greeter.git
  taskRef:
    kind: Task
    name: build-app
  workspaces:
    - emptyDir: {}
      name: source  
```

## usando o tekton para entregas de apps
- crie um serviceaccount
```
kubectl create serviceaccount tekton-deployer-sa
```
- vamos dar permissão para implantar o app no k8s, para isso devemos criar uma role (papel com permissoes)
- depois criar um rolebinding para vincular a role ao service account
- por fim criar a task para implantar (todos os arquivos encontram-se  no path tekton/entrega)

## pipeline com tekton
- primeiro precisamos instalar alguns recursos (git clone, buildah, kubernetes actions), como:
```
tkn hub install task git-clone
tkn hub install task maven
tkn hub install task buildah
tkn hub install task kubernetes-actions
```

- para pipeline, segue:
```
crie o secret com as credenciais do registry
vincule ao service account
execute os arquivos role, role-binding e tekton-greeter-pipeline-hub.yml
por fim execute o init.sh, para dar start a pipeline
```

# ARGOCD
- uma ferramenta de entrega contínua declarativa para k8s
- ela monitora os manifestos de um repositorio e diante configuração, os aplicam no cluster k8s
- exemplo:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/fabriciolfj/gitops.git
    path: ch07/bgd
    targetRevision: main
  syncPolicy:
    automated: # qualquer mudançca no git reflete no k8s
      prune: true #se remover o git, remove o app do k8s
      selfHeal: true  # se mudar alguma coisa na app direto no k8s, ele reimplanta o que está no git
```
- argocd pode monitorar um registry, ver que uma nova imagem foi gerada da app, atualizar o manifesto com esta, e consequentemente atualizar o k8s
- para isso precisamos instalar:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```
- para o argocd enviar a atualização da imagem ao repositorio git, aonde encontra-se os manifestos, precisamos de um secret com o token do git
```
kubectl -n argocd create secret generic git-creds --from-literal=username=test --from-literal=password=t232w2s
```