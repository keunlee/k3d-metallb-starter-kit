# Overview

Setup and Teardown a local Kubernetes Cluster with a Load Balancer, so that you can deploy to a local environment for local development. 

# Prerequisites

- docker - https://docs.docker.com/get-docker/
- k3d - (v4.4.6) - https://github.com/rancher/k3d/releases
- jq - https://stedolan.github.io/jq/
- kubectls - https://kubernetes.io/docs/tasks/tools/

# Setup

Create the Cluster and validate it's creation: 

```bash
# create the k3d cluster
k3d cluster create local-k8s --servers 1 --agents 3 --k3s-server-arg --no-deploy=traefik --wait

# set kubeconfig to access the k8s context
export KUBECONFIG=$(k3d kubeconfig write local-k8s)

# validate the cluster master and worker nodes
kubectl get nodes
```

Deploy the Load Balancer:

```bash
# determine loadbalancer ingress range
cidr_block=$(docker network inspect k3d-local-k8s | jq '.[0].IPAM.Config[0].Subnet' | tr -d '"')
cidr_base_addr=${cidr_block%???}
ingress_first_addr=$(echo $cidr_base_addr | awk -F'.' '{print $1,$2,255,0}' OFS='.')
ingress_last_addr=$(echo $cidr_base_addr | awk -F'.' '{print $1,$2,255,255}' OFS='.')
ingress_range=$ingress_first_addr-$ingress_last_addr

# deploy metallb 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml

# configure metallb ingress address range
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - $ingress_range
EOF
```

# Validation

Create an Nginx test deployment and expose via a Load Balancer. If the Load Balancer is working correctly, an external ip address should be assigned by Metallb.

```bash
# create a deployment (i.e. nginx)
kubectl create deployment nginx --image=nginx

# expose the deployments using a LoadBalancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# obtain the ingress external ip
external_ip=$(k get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# test the loadbalancer external ip
curl $external_ip
```

Expected Output:

```bash
# expected output: 

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# Teardown

Destroy the cluster

```bash
k3d cluster delete local-k8s
```