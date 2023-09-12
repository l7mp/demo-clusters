# K3s install reference

This guide provides a reference on how we installed `k3s` on the single VM that is running in Contabo.
K3s is perfect to use in for a single node Kubernetes cluster since it is very easy to install and consumes minimal resources.

However, in this environment we'll use `k3s` in a rather complicated setup (mainly for fun):
 - we didn't install the built-in `traefik` ingress, we use `nginx` instead (that is the straightforward part)
 - we don't use the build-in `flannel` CNI networking, we installed `cilium` instead, moreover we'll use the `kube-proxy` replacement feature for accelerated networking (more on the [here](https://cilium.io/use-cases/kube-proxy/))

 ## Install K3s with Cilium

We'll install `k3s` without:
 - `traefik` (we'll install `nginx`)
 - `flannel` (`cilium` will be the CNI plugin) 
 - `kube-proxy` (`cilium` will do this feature)
 - `network policy` (`cilium` will do this as well)

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--disable traefik --flannel-backend=none --disable-kube-proxy --disable-network-policy' sh -
```

It should take `k3s` about 30s to install. After it add the following housekeeping commands (just bash completion, and have the kube config file in the right place):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

kubectl version
kubectl get nodes && kubectl get pods -A
```

Notice, that the node is currently in `NotReady` status, since there is no CNI installed yet.

Download `helm` and add the `cilium` repo:
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
echo 'source <(helm completion bash)' >>~/.bashrc
helm repo add cilium https://helm.cilium.io/
helm repo update
```

Install `cilium` with the `kube-proxy` replacement feature:
```
export IP=$(ip -f inet addr show eth0 | grep -Po 'inet \K[\d.]+')
echo $IP
helm install cilium cilium/cilium --version 1.14.1  --namespace kube-system --set tunnel=disabled --set autoDirectNodeRoutes=true --set kubeProxyReplacement=strict --set loadBalancer.mode=dsr --set k8sServiceHost=$IP --set k8sServicePort=6443 --set operator.replicas=1 --set ipv4NativeRoutingCIDR=10.0.0.0/8
```

Check the `cilium` pods in the `kube-system` namespace:
```
kubectl get pods -n kube-system
```

As soon as they are in `Running` status, the all pods and the node should become in `Running` status very soon.

There you go, you have a single node but fully fledged Kubernetes cluster with accelerated networking.

## Install ArgoCD

This is basically the same guide as in the main readme, the only differences are i) the DNS name of the cluster, and ii) the folder we'll point ArgoCD in our git repo.

For reference, we use the following `values.yaml` file (fill it with your own domain):
```
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    https: true
    annotations: 
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    hosts: 
      - argocd.contabo-eu.stunner.cc
    tls: 
    - hosts:
      - argocd.contabo-eu.stunner.cc
      secretName: argocd-tls
```

Install `ArgoCD` with this override file:
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install -n argocd --create-namespace  -f values.yaml argocd argo/argo-cd
```

Step 3: add the ArgoCD `Application` that will install everything else.
The following is the content of the `argocd-applications.yaml` file:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-applications
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default
  source:
    directory:
      jsonnet: {}
      recurse: true
    path: ./contabo-eu/argocd-applications
    repoURL: https://github.com/l7mp/demo-clusters.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Apply this manifest and wait for ArgoCD to sync everything:
```
kubectl apply -f argocd-applications.yaml
```

