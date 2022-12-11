# Introduction

This document describes the installation and configuration of a k3s cluster on an Ubuntu server in the local network. The k3s cluster provides a lightweight Kubernetes environment for running and managing containerized applications.

To set up and run the k3s cluster, the following requirements must be met:

- The host machine must be running Ubuntu.
- Docker must be installed on the host machine. This is necessary to run the Kubernetes containers.
- The uwf service must be disabled on the host machine. This is necessary to avoid potential conflicts with the k3s cluster.

Once these requirements are satisfied, you can proceed with the installation and configuration of the k3s cluster.

## Describetion of current netowrk envrioment.

The k3s cluster is hosted on an Ubuntu server in the local network. The IP address of the host machine is "192.x.x.88", and the public IP of the network is "xxx.xxx.xxx.xxx".

## pre-installation check

After got access to the ubuntu server check local ip by run `ip a` and look for "enp3s0", which going to host the k3s cluster. Check port 80 is not blocked and accessable from other machines, by visiting the host ip from other computer on the same network.

1. On host machine, start simple http server by Run: `python -m SimpleHTTPServer 80`

2. On other computer run `curl -v 192.x.x.88`, expecting 200 response code
   - if port 80 is not accessble from other machine, troubleshoot the host machine firewall and routing table.

## installation

To install k3s, some network options are available from k3s document site. This example using `--tls-san` which allows you add additional hostnames or IPv4/IPv6 addresses as Subject Alternative Names on the TLS cert.

1. Run: `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--tls-san xxx.xxx.xxx.xxx" sh -`.

   - replace xxx.xxx.xxx.xxx with real public ip address
   - More options visit: https://docs.k3s.io/installation/configuration
   - Checkout the config file https://docs.k3s.io/installation/configuration#configuration-file
   - Installation document: https://docs.k3s.io/quick-start

2. Get Access:

   - Export access config file to user directory by run: `sudo k3s kubectl config view --raw > ~/.kube/config`
   - `chmod 600 ~/.kube/config`
   - `export KUBECONFIG=~/.kube/config` or add KUBECONFIG in the ~/.bashrc file so this env variable get load everytime when terminal start.

3. Access the cluster remotely:
   - Copy content out from `sudo k3s kubectl config view --raw`
   - Replace server: https://127.0.0.1:6443 with public ip address.

## post installation test

k3s installation come with traefik ingress controller. After k3s installed using other computer on the network visit defautl k3s ig endpoint should return 404 page not found. Also should using `kubectl get nodes` to test if cluster is up and running.

## Expose cluster to the internet

Config route

1. goto router's admin page
2. find port-forward
3. add rules for 22, 80 and 6443

## docker build example

Go to node-server folder and build a node web api and push the image

## Example Deployments

Node servers runing web api. this stack contains deployment and service, ingress for deploy a node web api using kubernetes.
Run: `kubectl apply -f deployments/node.yaml`

Nginx server running website. This stack contains deployment and service, ingress for a nginx servers.

1. Run: 'kubectl apply -f deployments/nginx.yaml`

## Config private registry

Registries: "/etc/rancher/k3s/registries.yaml" create file and use below config for registry access with auth. Restart k3s requires after mirror config changes. Run `systemctl restart k3s`

e.g mirror container registries

```
mirrors:
  docker.io:
    endpoint:
      - "https://registry-1.docker.io"
configs:
  "registry-1.docker.io":
    auth:
      username: xxxxx # this is the registry username
      password: xxxxx # this is the registry password
```

## Some other k3s file locations:

- k3s config file: "/etc/rancher/k3s/config.yaml"
- Kube access: "/etc/rancher/k3s/k3s.yaml"

## clean up

Run `/usr/local/bin/k3s-uninstall.sh` to delete k3s.

## some useful kubectl cli:

- `kubectl get events --sort-by='.metadata.creationTimestamp' -A -w`
- `kubectl get service`
