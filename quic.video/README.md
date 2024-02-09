# Attempt to run Media Over QUIC in Kubernetes

This repo contains the manifests to run the [quic.video](https://quic.video) website by [Kixelated](https://github.com/kixelated) in Kubernetes. The original site runs in GCP on VMs, you can check the Terraform code [here](https://github.com/kixelated/quic.video/tree/main/infra).

## Prerequisits

You can use any Kubernetes cluster for deployment which has an ingress controller (for managing http traffic), and [cert manager](https://cert-manager.io/) (for generating TLS certs) installed.

Here's a quick sippet to install these components:
```
# install nginx ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
# create a public loadbalancer IP for the ingress controller
kubectl patch service -n ingress-nginx ingress-nginx-controller -p '{"spec":{"type": "LoadBalancer"}}'
# check your public IP, usually this needs to be registered to your DNS provider
kubectl get service -n ingress-nginx ingress-nginx-controller -o jsonpath={.status.loadBalancer.ingress[0].ip}

# install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.2/cert-manager.yaml
# create a cluster issuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <your're email>
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Run MOQ test


