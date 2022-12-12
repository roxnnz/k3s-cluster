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

The node.yaml definition file includes some sample values for configuring cert-manager. To use cert-manager with your own settings, you will need to replace these sample values with real values that are appropriate for your environment. This will ensure that cert-manager is properly configured and can communicate with the certificate authorities and other components of your Kubernetes cluster.

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

## Cert-manager

Cert-manager is a Kubernetes controller that helps manage TLS certificates in a cluster. It can automatically provision and renew certificates from various certificate authorities, including Let's Encrypt, and integrates with ingress controllers to provide automatic TLS termination. This makes it easier to secure the traffic to your applications running on Kubernetes.

Waht is CRD?
In Kubernetes, a Custom Resource Definition (CRD) is a way to extend the Kubernetes API to create custom resources that can be used in your Kubernetes cluster. CRDs allow you to define new API objects that can be used in the same way as built-in Kubernetes objects, such as pods and deployments. This enables you to add new functionality to Kubernetes without having to modify the core Kubernetes code. For example, you could use CRDs to define a new type of resource called a "foo" that represents some custom functionality that your applications need. Then you could use the Kubernetes API to create, manage, and delete "foo" resources in your cluster.

To install cert-manager and config tls:

1. Add cert-manager helm chart and update
2. Install cert-manager CRDs (Custom Resource Definition) by running kubectl apply `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.1/cert-manager.crds.yaml`
3. Create namespace for cert-manager by `kubectl create namespace cert-manager`
4. helm install cert-manager `helm install cert-manager --namespace cert-manager --version v1.10.1 jetstack/cert-manager`
5. Check cert-manager installed correctly
6. Under cert-manager namespace, install TLS issuers for prod by run: `kubectl apply -f issuer.yaml`. Need to create issuer.yaml file like below: (also recommand to install dev issuer for testing)
   - e.g
     ```
     apiVersion: cert-manager.io/v1
     kind: Issuer
     metadata:
     name: letsencrypt-prod
     spec:
     acme:
        # The ACME server URL
        server: https://acme-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: xxxxxxxxxx@gmail.com
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
        name: letsencrypt-prod
        # Enable the HTTP-01 challenge provider
        solvers:
        - http01:
           ingress:
              class: traefik
     ```
   - It is also recommended to install the dev issuer for testing purposes. This will allow you to easily test and debug your TLS certificates in a development environment without having to use a real certificate authority.
7. Check if issuer has been installed correctly
8. Add configs in ingress defination file for:
   - add annotaion: `cert-manager.io/issuer: "letsencrypt-prod"`
   - add tls describetions under spec:
     e.g
     ```
     spec:
        tls:
        - hosts:
              - example.domain.com
           secretName: node-service-tls
        rules: ...
     ```
   - Note that the name of the certificate in the Secret resource created by cert-manager will be the same as the secretName defined in your certificate definition. This is the name that you will use to reference the certificate in your ingress resource or other components of your Kubernetes application.
9. A pod will be started at the this namespace, it's for checking webhook call for acma check. after check finished cert in ready status:

```
$ kubectl get certificate -n ng
NAME READY SECRET AGE
node-service-tls False node-service-tls 26s

$ kubectl get certificate -n bookings
NAME READY SECRET AGE
node-service-tls False node-service-tls 27s

$ kubectl get certificate -n bookings
NAME READY SECRET AGE
node-service-tls True node-service-tls 28s

$ kubectl get certificate -n bookings
NAME READY SECRET AGE
node-service-tls True node-service-tls 29s
```

10. some useful links:

- https://cert-manager.io/docs/installation/helm/
- https://cert-manager.io/docs/tutorials/acme/nginx-ingress/
