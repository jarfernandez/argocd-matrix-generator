# Argo CD Matrix Generator

Prueba del [Matrix Generator de Argo CD](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Matrix) para instalar desde un cluster Kubernetes [kind](https://kind.sigs.k8s.io/) el [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/) en otros dos clusters Kubernetes kind. Para ello, se utiliza un [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) con el Matrix Generator, que a su vez combina el [Git Generator](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Git/) con el [Cluster Generator](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Cluster/).

Lo primero que necesitamos hacer es crear los 3 clusters Kubernetes con kind:

```bash
kind create cluster --name argocd
kind create cluster --name dev
kind create cluster --name prod
```

En el primero de los clusters instalaremos [Argo CD](https://argoproj.github.io/cd/) y en los otros dos desplegaremos el NGINX Ingress Controller.

El siguiente paso es instalar Argo CD en el primer cluster:

```bash
kubectl config use-context kind-argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

Una vez instalado Argo CD, hacemos port forward del servicio para poder acceder a la interfaz web:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Este último comando nos permitirá acceder a la interfaz web a través de https://localhost:8080.

Para acceder a la interfaz web, el usuario es `admin` y la contraseña la podemos obtener con el siguiente comando:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -ojsonpath="{.data.password}" | base64 -d ; echo
```

También necesitaremos [instalar el CLI de Argo CD](https://argo-cd.readthedocs.io/en/stable/cli_installation/). La forma más sencilla de instalarlo es utilizando [Homebrew](https://brew.sh/):

```bash
brew install argocd
```

Utilizaremos el CLI de Argo CD para dar de alta los clusters remotos dev y prod en los que queremos desplegar el NGINX Ingress Controller:

```bash
kubectl config set-context --current --namespace argocd
argocd cluster add kind-dev --core --label environment=dev --name dev --yes
argocd cluster add kind-prod --core --label environment=prod --name prod --yes
```

El siguiente paso es parchear los secretos de los clusters remotos dev y prod para que sean accesibles desde Argo CD. Para ello ejecutaremos los siguientes comandos:

```bash
secret=$(kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster --no-headers | awk '{ print $1 }' | head -1)
env=$(kubectl get secret -n argocd $secret -ojsonpath="{.data.name}" | base64 -d ; echo)
server=$(echo -n "https://$(kubectl get endpoints --context kind-$env -n default --no-headers | awk '{ print $2 }')" | base64)
kubectl patch secret $secret --type='json' -p='[{"op" : "replace" ,"path" : "/data/server" ,"value" : "'$server'"}]'
secret=$(kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster --no-headers | awk '{ print $1 }' | tail -1)
env=$(kubectl get secret -n argocd $secret -ojsonpath="{.data.name}" | base64 -d ; echo)
server=$(echo -n "https://$(kubectl get endpoints --context kind-$env -n default --no-headers | awk '{ print $2 }')" | base64)
kubectl patch secret $secret --type='json' -p='[{"op" : "replace" ,"path" : "/data/server" ,"value" : "'$server'"}]'
```

Por último, generaremos las aplicaciones en los clusters remotos ejecutando el siguiente comando:

```bash
kubectl apply -f nginx-ingress-application-set.yaml
```

Podemos ver el estado de las aplicaciones en la interfaz web o con el siguiente comando:

```bash
kubectl get applications -n argocd
```
