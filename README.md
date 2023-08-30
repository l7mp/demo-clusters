# GitOps repository for public Stunner demos

This is a GitOps repository to showcase the capabilities of Stunner in various cloud platforms.
We use [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) to deploy the manifests in this repo into the Kubernetes clusters.

## Google Cloud Platform

The following environment was started in GCP:
 - Type: GKE standard cluster
 - Kubernetes version: v1.27.3-gke.100
 - Region: europe-central2 (Warsaw, Poland)
 - Nodes: 1 x e2-standard-4 (4 CPU, 16GB memory)

 Runnig applications:
  - `LiveKit`: [livekit.gcp-europe-central2.stunner.cc](livekit.gcp-europe-central2.stunner.cc)
  - `Jitsi`: [jitsi.gcp-europe-central2.stunner.cc](jitsi.gcp-europe-central2.stunner.cc)
  - `Edumeet`: [edumeet.gcp-europe-central2.stunner.cc](edumeet.gcp-europe-central2.stunner.cc)
  - `Neko`: [neko.gcp-europe-central2.stunner.cc](neko.gcp-europe-central2.stunner.cc)
  - `Cloud Retro`: [cloudretro.gcp-europe-central2.stunner.cc](cloudretro.gcp-europe-central2.stunner.cc)
  - `Kurento Magic Mirror`: [magicmirror.gcp-europe-central2.stunner.cc](magicmirror.gcp-europe-central2.stunner.cc)
  - `Kurento one to one call`: [one2one.gcp-europe-central2.stunner.cc](one2one.gcp-europe-central2.stunner.cc)

## ArgoCD reference

The following is a guide on how start ArgoCD on a cluster, so might copy this workflow.
ArgoCD has to be bootstraped, and point back to a `git` repository.
For reference, we'll use the files in the `gcp-europe-central2` folder.

Step 1: if you don't have `Helm`, install it:
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Step 2: Install `ArgoCD` using `Helm`.
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
      - argocd.gcp-europe-central2.stunner.cc
    tls: 
    - hosts:
      - argocd.gcp-europe-central2.stunner.cc
      secretName: argocd-tls
```

Install `ArgoCD` with this override file:
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install -n argocd --create-namespace  -f values.yaml argocd argo/argo-cd
```

Step 3: add the ArgoCD `Application` that will read install everything else.
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
    path: ./gcp-europe-central2/argocd-applications
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

Step 4: Set-up DNS records.
In this installation ArgoCD will install an `Nginx Ingress Controller` that will be exposed to the Internet via a `LoadBalacer` type Kubernetes service. In order to reach your applications you have to register that `ExternalIP` to you DNS provider. The following command will show you the public IP address of the ingress controller (it might take a few minutes until your cloud provider provisions a public IP):
```
kubectl get service -n ingress-nginx ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

We took this public IP and register an `A` record for `*.argocd.gcp-europe-central2.stunner.cc` in our DNS provider.

If that is done, you should be able to access the ArgoCD UI in your domain (in our case `argocd.gcp-europe-central2.stunner.cc`). The user is `admin`, and you get the password with the following command:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
``` 