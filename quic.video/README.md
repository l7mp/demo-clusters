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

## Notes

Our current test environment ([moq.stunner.cc](https://moq.stunner.cc)) runs on a single VM in GCP using [k3s](https://k3s.io). `k3s` uses a [load balancer](https://docs.k3s.io/networking#service-load-balancer) which just opens the same ports of the VMs that the Kubernetes `services` use. Hence, the public IP of the `Nginx Ingress` (for regular HTTP/HTTPS TCP/80,443) and the `relay` (for HTTP3 UDP/443) will be the same.

In a cloud managed Kubernetes you'll typically get different public IPs for those two services. This usually won't be a problem, you just have to register a different IP in your DNS for the `realy`. However, the `certificate` object for the `relay` domain is currently set up to use the [HTTP challange of Let's Encrypt](https://cert-manager.io/docs/configuration/acme/http01/). Since this uses normal HTTP portocol at `TCP/80`, that traffic will go to the `relay` domain service and be dropped, thus you won't get a valid certificate. In this case we suggest to use [DNS challange](https://cert-manager.io/docs/configuration/acme/dns01/) insted.

Another solution would be to use the `Nginx Ingress` to expose the `relay` service. You can find a guide [here](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/). This way, every service will be exposed under the same public IP, thus this will solve the issue with the HTTP challange, since the ingress can handle the HTTP connection for the challange, and forward the QUIC traffic to the `relay`. However, in this scenario the `ingress-nginx-controller` service would have to listen in both TCP (80, 443) and UDP (443) protocol, which is called a `mixed protocol service`. This [feature](https://github.com/kubernetes/enhancements/issues/1435) is only GA since Kubernetes 1.28, so if you're using an older Kubernetes version you're simple unable to create such service with `type: LoadBalancer`. After 1.28 you can, but still many cloud managed Kubernetes environment does not support this. E.g. I got the following error in GKE (env: Kubernetes version: v1.29.0-gke.1381000, tested in 2024 February):
```
Warning  SyncLoadBalancerFailed                 8s (x3 over 23s)  service-controller  Error syncing load balancer: failed to ensure load balancer: mixed protocol is not supported for LoadBalancer
```

