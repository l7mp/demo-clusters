# GitOps repository for public Stunner demos

This is a GitOps repository to showcase the capabilities of Stunner in various cloud platforms.
We use [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) to deploy the manifests in this repo into the Kubernetes clusters.

## Google Cloud Platform - Standard GKE

The following environment was started in GCP:
 - Type: GKE standard cluster
 - Kubernetes version: v1.27.3-gke.100
 - Region: europe-central2 (Warsaw, Poland)
 - Nodes: 6 x e2-medium (2 CPU, 4GB memory per VM)

 Runnig applications:
  - `LiveKit`: [livekit.gcp-europe-central2.stunner.cc](https://livekit.gcp-europe-central2.stunner.cc)
  - `Janus`: [janus.gcp-europe-central2.stunner.cc](https://janus.gcp-europe-central2.stunner.cc)
  - `Jitsi`: [jitsi.gcp-europe-central2.stunner.cc](https://jitsi.gcp-europe-central2.stunner.cc)
  - `Elixir WebRTC`: [nexus.gcp-europe-central2.stunner.cc](https://nexus.gcp-europe-central2.stunner.cc)
  - `Edumeet`: [edumeet.gcp-europe-central2.stunner.cc](https://edumeet.gcp-europe-central2.stunner.cc)
  - `Neko`: [neko.gcp-europe-central2.stunner.cc](https://neko.gcp-europe-central2.stunner.cc)
  - `Cloud Retro`: [cloudretro.gcp-europe-central2.stunner.cc](https://cloudretro.gcp-europe-central2.stunner.cc)
  - `Kurento Magic Mirror`: [magicmirror.gcp-europe-central2.stunner.cc](https://magicmirror.gcp-europe-central2.stunner.cc)
  - `Kurento one to one call`: [one2one.gcp-europe-central2.stunner.cc](https://one2one.gcp-europe-central2.stunner.cc)

Notes on the environment: in this scenario we only run one `stunnerd` which distributes the traffic amoung all the running WebRTC applications. To that end, we only have one `GatewayClass`, `GatewayConfig` and `Gateway` objects (see [here](gcp-europe-central2/argocd-applications/stunner-common.yaml)), and all the applicaitons have their specific `UDPRoute` object (see files in `./gcp-europe-central2/apps/<app name>/udp-route.yaml`).

Only two service are exposed to the internet: `ingress-nginx-controller` for HTTP(S) traffic, and `stunnerd` for STUN/TURN traffic to handle WebRTC connections:
```
kubectl get services -A | grep LoadBalancer
NAMESPACE       NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                        AGE           
ingress-nginx   ingress-nginx-controller   LoadBalancer   10.36.5.172    34.118.56.177    80:32588/TCP,443:31220/TCP     14h
stunner         udp-gateway                LoadBalancer   10.36.6.102    34.116.153.145   3478:31073/UDP                 12h
```

Thus we added the following entries to our DNS provider:
| Type | Hostame                          | Content        |
|------|----------------------------------|----------------|
| A    | gcp-europe-central2.stunner.cc   | 34.116.153.145 |
| A    | *.gcp-europe-central2.stunner.cc | 34.118.56.177  |

You can also examine that all WebRTC applications are configured to use `gcp-europe-central2.stunner.cc:3478` as their STUN/TURN server, using `relay` mode in the TURN setup.

Also, `Jitsi` does not work in this environment, since we use `plaintext` credentials in the Stunner config, which Jitsi does not support. Please examine another environment for `Jitsi`, where we set up separate Stunner gateways for every app, and the one for `Jitsi` will be configured for `longterm` credentials.

## Contabo VPS

The following environment was started in Contabo:
 - Type: Single virtual machine with k3s
 - Kubernetes version: v1.27.5+k3s1
 - Region: EU (Nuremberg, Germany)
 - Nodes: 1 x CLOUD VPS M (6 CPU, 16GB memory)

 Runnig applications:
  - `LiveKit`: [livekit.contabo-eu.stunner.cc](https://livekit.contabo-eu.stunner.cc)
  - `Jitsi`: [jitsi.contabo-eu.stunner.cc](https://jitsi.contabo-eu.stunner.cc)
  - `Edumeet`: [edumeet.contabo-eu.stunner.cc](https://edumeet.contabo-eu.stunner.cc)
  - `Neko`: [neko.contabo-eu.stunner.cc](https://neko.contabo-eu.stunner.cc)
  - `Cloud Retro`: [cloudretro.contabo-eu.stunner.cc](https://cloudretro.contabo-eu.stunner.cc)
  - `Kurento Magic Mirror`: [magicmirror.contabo-eu.stunner.cc](https://magicmirror.contabo-eu.stunner.cc)
  - `Kurento one to one call`: [one2one.contabo-eu.stunner.cc](https://one2one.contabo-eu.stunner.cc)

Notes on the environment: in this scenario we run a separate `stunnerd` for every application with their own `GatewayClass`, `GatewayConfig`, `Gateway` and `UDPRoute` objects (see files `contabo-eu/apps/<app name>/stunnerd.yaml` for the Helm install, and `contabo-eu/apps/<app name>/stunner-gateway.yaml` for the other objects). To that end, every application can have a different `GatewayConfig` with different credentials. This way for e.g., Jitsi can use `longterm` credentials, so here it does work. Also notice that we defined different UDP listening ports for every application. Thoretically this will create a separate `LoadBalancer` type service for every application, but in `k3s` the [built-in load balancer](https://docs.k3s.io/networking#service-load-balancer) will just assign the IP of the VM for every exposed service:

```
kubectl get services -A | grep LoadBalancer
NAMESPACE       NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                        AGE           
ingress-nginx   ingress-nginx-controller   LoadBalancer   10.36.5.172    34.118.56.177    80:32588/TCP,443:31220/TCP     14h
stunner         udp-gateway                LoadBalancer   10.36.6.102    34.116.153.145   3478:31073/UDP                 12h
```

In our DNS provider we added this public IP (the public IP of the VM itself) for the given entries:
| Type | Hostame                 | Content       |
|------|-------------------------|---------------|
| A    | contabo-eu.stunner.cc   | 144.91.97.105 |
| A    | *.contabo-eu.stunner.cc | 144.91.97.105 |

## ArgoCD reference

The following is a guide on how to start ArgoCD on a cluster, so you might copy this workflow.
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
In this installation ArgoCD will install an `Nginx Ingress Controller` that will be exposed to the Internet via a `LoadBalacer` type Kubernetes service. In order to reach your applications you have to register that `ExternalIP` to your DNS provider. The following command will show you the public IP address of the ingress controller (it might take a few minutes until your cloud provider provisions a public IP):
```
kubectl get service -n ingress-nginx ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

We took this public IP and registered an `A` record for `*.argocd.gcp-europe-central2.stunner.cc` in our DNS provider.

If that is done, you should be able to access the ArgoCD UI in your domain (in our case `argocd.gcp-europe-central2.stunner.cc`). The user is `admin`, and can you get the password with the following command:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
``` 
