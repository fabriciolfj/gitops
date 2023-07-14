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