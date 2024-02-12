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

Normally, you just have to apply the manifests in this folder:
```
kubectl apply -f .
```

The `relay` component will listen in `UDP/443` for `QUIC/WebTransport` connection, so the service
that expose the `relay` pod has type `LoadBalancer`. You can check the public IP you got from you provider
with the following command:
```
kubectl get service relay
```

There are also two services exposed via `ingress`: `moq-api` and `moq-web`.
In this example we used the following domains:
| Hostame              | IP address   | Note              |
|----------------------|--------------|-------------------|
| moq.stunner.cc       | 34.116.151.9 | web frontend [repo](https://github.com/kixelated/moq-js) |
| api.moq.stunner.cc   | 34.116.151.9 | API server   [repo](https://github.com/kixelated/moq-rs/tree/main/moq-api) |
| relay.moq.stunner.cc | 34.116.151.9 | WebTransport relay server [repo](https://github.com/kixelated/moq-rs/tree/main/moq-relay) |

If you want to use you're own domain, the following command will replace `moq.stunner.cc` to `example.com` 
(using [kustomize](https://github.com/kixelated/moq-rs/tree/main/moq-relay) would be more elegant, but for now `sed` will do):
```
sed -i 's/moq.stunner.cc/example.com/g' *
```

