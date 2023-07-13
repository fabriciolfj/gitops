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
