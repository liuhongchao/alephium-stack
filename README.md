Alephium Stack
==============

Code in this repository helps to run the
[Alephium](https://github.com/alephium/alephium) stack on [Google
Cloud Platform](https://cloud.google.com/) using [Kubernetes
Engine](https://cloud.google.com/kubernetes-engine/). The following
sites are setup and exposed using the code in this repository:

- [Alephium Explorer](https://alephium.hongchao.me/#/blocks) (`https://alephium.hongchao.me`)
- [Alephium Overview - Grafana](https://grafana.hongchao.me/d/S3eJTo3Mk/alephium-overview?orgId=1&refresh=10s) (`https://grafana.hongchao.me`)
- [Alephium API Documentation](https://alephium.hongchao.me/docs) (`https://alephium.hongchao.me/docs`)
- Alephium Full Node with [CPU Miner](https://github.com/alephium/cpu-miner) Enabled (`35.241.179.18:9973`)

[Terraform](terraform) files are GCP specific. [Kubernetes YAML
files](kubernetes) can probably be re-used with other cloud providers.

## Prerequisit

* [Terraform](https://www.terraform.io/)
* [Google Cloud SDK](https://cloud.google.com/sdk/)

## Setup

### GCP project and GKE cluster

1. If you have not setup GCP for your google account already, you can
   try to set it up [here](https://cloud.google.com/gcp/). For new
   users, Google offers 300 USD credits. To start the trial, Google
   requires registration of payment method. According to the [terms
   and conditions](https://cloud.google.com/terms/free-trial/) for GCP
   free trial, mining cryptocurrency is not allowed, but running a
   full node is *not* mining. Outside of the trial period however, it
   is ok to mine cryptocurrency.
2. After GCP is setup, you can find your billing account id
   [here](https://console.cloud.google.com/billing)

After that, go to the [terraform](terraform) folder and run

```
terraform init
terraform apply -var="project_billing_account=YOUR_BILLING_ACCOUNT"
```

More powerful [machine
types](https://cloud.google.com/compute/docs/machine-types) can be set
up using `kubernetes_node_pool_machine_type` variable, which might be
helpful during mining. More available variables please check the
[variables.tf](terraform/variables.tf) file.

### Kubernetes Resources

#### Alephium
Go to the [kubernetes](kubernetes) folder and run

```
kubectl apply -k alephium
```

This will install the Alephium full node, block explorer as well as
the CPU miner in the `alephium` namespace.

#### Monitoring (Optional)
Go to the [kubernetes](kubernetes) folder and run

```
kubectl apply -k monitoring
```

This will install [Prometheus](https://prometheus.io/) and
[Grafana](https://grafana.com/) in the `monitoring`
namespace. Prometheus server is configured to scrape the metrics
endpoint for Alephium full node.

#### Exposing Sites (Optional)

First we need to install

- [cert manager](https://cert-manager.io/docs/),  which automatically
  manages certificates using letsencrypt.
- [ingress-nginx](https://github.com/kubernetes/ingress-nginx) which
  exposes Kubernetes services using Nginx as reverse proxy and load
  balancer.

Go to the [kubernetes](kubernetes) folder and run

```
kubectl apply -k cert-manager
kubectl apply -k ingress-nginx
```

After that, to expose the sites mentioned at the beginning of the
README, run

```
# exposes alephium.hongchao.me and alephium.hongchao.me/docs
kubectl apply -f alephium/alephium-ingress.yaml

# exposes grafana.hongchao.me
kubectl apply -f monitoring/grafana-ingress.yaml
```

Please take a look at
[alephium-ingress.yaml](kubernetes/alephium/alephium-ingress.yaml) and
[grafana-ingress.yaml](kubernetes/monitoring/grafana-ingress.yaml),
update with the new domain accordingly. Note that you also need to
create a DNS A record for your domain to point to the external IP address of
the `ingress-nginx` service.

```
➜ kubectl get service ingress-nginx-controller --namespace ingress-nginx
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.3.240.206   34.76.236.163   80:32611/TCP,443:31303/TCP   3h37m
```

Here the external IP is `34.76.236.163`, which is the same as
`alephium.hongchao.me` as shown below:

```
➜ kubernetes git:(master) ✗ nslookup alephium.hongchao.me
Server:         192.168.1.1
Address:        192.168.1.1#53

Non-authoritative answer:
Name:   alephium.hongchao.me
Address: 34.76.236.163
```

## Usage

- Run Alephium stack in K8S and expose to the public
- Mining with powerful machines in the cloud
- Develop against remote Alephium locally, e.g.

```
# Port forward alephium from Kubernetes cluster
➜ kubectl port-forward svc/alephium 12973
Forwarding from 127.0.0.1:12973 -> 12973
Forwarding from [::1]:12973 -> 12973

# Curl locally
➜ curl localhost:12973/infos/self-clique
{"cliqueId":"03764042ab3c875481e5eed6d7a59027a7232582c189e1c266f57b62591ae0d8e0","networkId":1,"numZerosAtLeastInHash":18,"nodes":[{"address":"127.0.0.1","restPort":12973,"wsPort":11973,"minerApiPort":10973}],"selfReady":true,"synced":true,"groupNumPerBroker":4,"groups":4}
```