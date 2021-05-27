# Kubernetes cluster in a bare metal server

Source articles:

* https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-ingress-guide-nginx-example.html
* https://itnext.io/bare-metal-kubernetes-with-kubeadm-nginx-ingress-controller-and-haproxy-bb0a7ef29d4e

Requirements:

* A machine (or VM) with access from the web and the permission to set inbound traffic for different ports.
* A FQND pointing to the machine's public IP with at least 2 subdomain (alpine and latest).

## Provision a VM (optional)

We can provision a VM to act as our bare metal machine. The commands below will create an Azure machine. You can skip this section or do the same using any other could service provider.

```shell
# Create SSH keys (optional)
$ ssh-keygen -t rsa

# Create group
$ az group create -n k8s-cluster -l brazilsouth

# Create VM
$ az vm create \
-n "vm" \
-g "k8s-cluster" \
-l "brazilsouth" \
--vnet-name "k8s-cluster-vnet" \
--subnet "default" \
--nsg "vm-nsg" \
--size "Standard_B2s" \
--image "UbuntuLTS" \
--admin-username "azureuser" \
--ssh-key-values ~/.ssh/id_rsa.pub \
--tags "deploy=cli"

# Write down the machine's IP: xxx.xxx.xxx.xxx

# Let's open some ports
$ az vm open-port --port 443 -n vm -g k8s-cluster --priority 500
$ az vm open-port --port 80  -n vm -g k8s-cluster --priority 501
$ az vm open-port --port 30080  -n vm -g k8s-cluster --priority 502
$ az vm open-port --port 30443  -n vm -g k8s-cluster --priority 503

# Login into the machine
$ ssh -i ~/.ssh/id_rsa azureuser@xxx.xxx.xxx.xxx
```

## Required software

Install Certbot, Docker, Compose, Kubernetes and Kubeadm.

```shell
(vm) $ source <(curl -s https://gist.githubusercontent.com/nu12/6d7813d71d195cfa2fcd0879010429d0/raw)

(vm) $ git clone https://github.com/nu12/bare-metal-kubernetes.git .
```

## Basic setup

Let's test our connection to the server with a simple application.

```shell
# To create the app
(vm) $ docker-compose -f 0-docker-compose/docker-compose.yml up

# To remove the app
(vm) $ docker-compose -f 0-docker-compose/docker-compose.yml down
```
Adjust `alpine.example.com` in 0-docker-compose/docker-compose.yml accordingly. The application should be available at port 80 using the FQND. You can remove the app after testing.

## Kubernetes setup

Now the we know that we can reach our application from the web, let's move it to a Kubernetes cluster.

```shell
# Start a Kubernetes cluster (master node)
(vm) $ sudo kubeadm init

# Write down the node join command: kubeadm join 10.0.0.4:6443 --token bui1cf.74e65jv1mxdn5hix --discovery-token-ca-cert-hash sha256:be35c9ce6683bf61b1fdd36998d26f9cb2552124413ccced00429a31fd80424f

# ConfiG kubectl to use the API.
(vm) $ mkdir -p $HOME/.kube
(vm) $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
(vm) $ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Setup cluster network
(vm) $ kubectl apply -f calico.yaml

# Remove master node taint
(vm) $ kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Node port
Let's test the application using Kubernetes with the commands below.

```shell
# To create the app
(vm) $ kubectl apply -f 1-zero-downtime-update/

# To remove the app
(vm) $ kubectl delete -f 1-zero-downtime-update/
```
The application should be available at port 30080 using the FQND. You can remove the app after testing.


### Nginx ingress

We don't want our application to be exposed through port 30080, we want it on ports 80 and 443. 

Let's use Kubernetes ingress to create a "reverse-proxy", allowing us to have multiple applications running in the cluster and use HAProxy to make them accessible via HTTP(S) requests.

```shell
(vm) $ kubectl create ns ingress-nginx

(vm) $ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml

(vm) $ kubectl apply -f deploy.yaml

(vm) $ rm deploy.yaml

(vm) $ kubectl get services -A

# Write down the ports mapped: 80:30080, 443:30443
```
Test ingress

```shell
# To create the app
(vm) $ kubectl apply -f 2-ingress/
```

Adjust `alpine.example.com` in 2-ingress/ingress.yaml accordingly. The application should be available at the mapped port using the FQND. This time we will not remove the application. Continue with the HAProxy configuration below.

### Using HAproxy to handle inblound HTTP and HTTPS traffic

Now we will make the application available on ports 80 and 443.

```shell

# Install HAProxy
(vm) $ sudo apt-get install -y haproxy

# Generate SSL certificate
(vm) $ sudo certbot certonly -d alpine.example.com -d latest.example.com
# Write down the path for the certificate: /etc/letsencrypt/live/alpine.example.com

# Configure /etc/haproxy/haproxy.cfg
(vm) $ sudo cat /etc/letsencrypt/live/alpine.example.com/cert.pem /etc/letsencrypt/live/alpine.example.com/privkey.pem > ssl.pem
(vm) $ sudo mv ssl.pem /etc/haproxy
(vm) $ cat /etc/haproxy/haproxy.cfg haproxy.cfg > full_haproxy.cfg

# Configure full_haproxy if needed. Check the private IP for each node. 

# Replace configuration with the full file
(vm) $ sudo mv full_haproxy.cfg /etc/haproxy/haproxy.cfg
# Restart the service
(vm) $ sudo service haproxy restart
```
You application is now available on port 80 and 443. HAProxy is configured to automatically redirect HTTP traffic to HTTPS and our Kubernetes ingress can send the traffic to different applications.

Success!

You may now disable access to any port other than 80, 433.

## Removing resources (optional)

In case you have created cloud resources, delete them now.

```shell
$ az group delete -n k8s-cluster --no-wait
```