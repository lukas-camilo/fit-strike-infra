# Como criar cluster kubenetes do zero

## Instale do Docker Desktop

https://www.docker.com/products/docker-desktop/

## Instale o Kind

Baixe o kind: 

`curl -Lo kind.exe https://kind.sigs.k8s.io/dl/v0.22.0/kind-windows-amd64`

Mova-o para a pasta de instalação: 

`move .\kind.exe C:\Windows\System32`

## Instale o Kukectl

Baixe o kubectl: 

`curl -LO "https://dl.k8s.io/release/v1.30.1/bin/windows/amd64/kubectl.exe`

Mova o para a pasta de instalação: 

`move .\kubectl.exe C:\Windows\32`

## Crie o cluster kubernetes

`kind create cluster --name cluster-k8s --config kind-config.yaml`

Verifique os dados de seus cluster:

`kubectl cluster-info --context kind-cluster-k8s`

`kubectl get nodes`

Caso queira remover o cluster use o comando abaixo:

`kind delete cluster --name cluster-k8s`

## Instale o argo cd

Crie um namespace para o argo:

`kubectl create namespace argocd`

Add o repositório do argocd via kubectl:

`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`

Valide se o addon foi incluido com sucesso, analise a coluna **READY** e **STATUS** = ***Running***:

`kubectl get pods -n argocd`

Exponha o Argo CD via port-forward para acessar a interface web (abra um novo terminal):

`kubectl port-forward svc/argocd-server -n argocd 8080:443`

`Acesse: http://localhost:8080`

Obtenha a senha inicial do usuário admin:

> ⚠️ **Atenção:** Recomendado alterar a senha ao realizar primeiro acesso, basta acessar user-info

`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

## Instale o ingress controller

Add o repositório do ingress via kubectl:

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/kind/deploy.yaml`

Valide se o addon foi incluido com sucesso, analise a coluna **READY**:

`kubectl get pods -n ingress-nginx`

> ⚠️ **Atenção:** Caso ao consultar o status dos pods o *controller* esteja com status pending possívelmente existe um erro relacionado aos labes do node cluster com o pod do ingress, para validar utilize o comando abaixo e verifique a mensagem de erro:
>
>`kubectl describe pod <nome-do-pod> -n ingress-nginx`
>
>Se o problema estiver relacionado a labels siga o passo a baixo para resolver.
>
>Consulte a lista de nodes do cluster: `kubectl get nodes`
>
>Adicione o label ao node: `kubectl label node <nome-do-node> ingress-ready=true`
>
>Verifique se o node está com o label corretamente: `kubectl get nodes --show-labels`
>
>Verifique se o pod controller está rodando: `kubectl get pods -n ingress-nginx`

Crie um arquivo chamado argocd-ingress.yaml com o seguinte conteúdo:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
    - host: argocd.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

Aplique o arquivo de configuração:

`kubectl apply -f argocd-ingress.yaml`

Adicione o host ao seu arquivo hosts, edite o arquivo `C:\Windows\System32\drivers\etc\hosts` e adicione:

`127.0.0.1 argocd.local`

Acesse o argocd pelo navegador:

`https://argocd.local`

## Instalar o Keda

Adicione o kedacore utilizando Helm:

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

Adicione também o add-on Keda HTTP para realizar o scalling baseado em métricas de requests HTTP:

`helm install http-add-on kedacore/keda-add-ons-http --namespace keda --create-namespace -f values.yaml`


> **Atenção:** Para que as aplicações façam o scalling corretamente será necessário criar o objeto ScaledObject apontando para o deployment da aplicação, abaixo um código de exemplo:
>
>```
>apiVersion: keda.sh/v1alpha1
>kind: ScaledObject
>metadata:
>  name: exemplo-scaledobject
>  namespace: default
>spec:
>  scaleTargetRef:
>    name: nome-do-seu-deployment
>  minReplicaCount: 0
>  maxReplicaCount: 10
>  triggers:
>    - type: cpu
>      metadata:
>        type: Utilization
>        value: "50"
>```

## Configurando a primeira aplicação no cluster

### Pré-requisitos

- Ter a imagem da aplicação publicada no Docker Hub

### Passo a passo

1. Criar o deployment.yaml da aplicação, abaixo segue um exemplo:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: minha-app
        image: seu-usuario-dockerhub/nome-da-imagem:tag
        ports:
        - containerPort: 80 # ajuste conforme sua aplicação
```

2. Criar o service.yaml para expor sua aplicação, abaixo segue um exemplo:

```
apiVersion: v1
kind: Service
metadata:
  name: minha-app-service
spec:
  type: NodePort
  selector:
    app: minha-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080 # porta externa, pode ajustar
```

3. Aplique ambos arquivos em seu cluster:

```
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml
kubectl apply -f scaled-object.yaml
```

4. Verifique se os PODs e Services estão rodando:

```
kubectl get pods
kubectl get svc
```

5. Para acessar a aplicação primeiro obtenha o IP do cluster:

```
kubectl get nodes -o wide
```

6. Acesse a aplicação via:

```
Acesse via: http://<IP-do-node>:30080
```