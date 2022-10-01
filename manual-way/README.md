# Getting Started with k3s in Homelab / Cloud 

[K3s](https://rancher.com/docs/k3s/latest/en/) is lightweight distribution of Kubernetes developed and maintained by [Rancher](https://rancher.com/).

## Prerequisites

### Setup Machines, OS and IPs

- Your nodes and servers should be running a Kubernetes compatible OS - I used Ubuntu Server Cloud Image 22.04 LTS. Make sure to use SSD/Nvme Drives for faster IOPS because of embedded etcd usage.

- Set a static IP address for the machines, I used DHCP reserved leases. eg:

        nginix-lb = 192.168.1.150 = 2GB
        k3s-machine-1 = 192.168.1.151 = 4GB
        k3s-machine-2 = 192.168.1.152 = 4GB
        k3s-machine-3 = 192.168.1.153 = 4GB

  - Make sure all nodes are fully updated, if you use Cloud Image with Cloud-Init it will auto upgrade all machines on first boot or run `sudo apt update && sudo apt upgrade -y` manually.

### Setup A Load Balancer For All Server Nodes in HA (I used NGINX upstream functionality)

- Spin up NGINX LXC or in Docker whatever you like - I used LXC and set its IP to `192.168.1.150`
- Modify the nginix.conf file located at `/etc/nginx/nginx.conf` to something like this replacing your IP addresses for machines.

      load_module /usr/lib/nginx/modules/ngx_stream_module.so;

      events {
      }

      stream {
          upstream api {
              server 192.168.1.151:6443; # these are your server/node IPs
              server 192.168.1.152:6443; # these are your server/node IPs
              server 192.168.1.153:6443; # these are your server/node IPs
          }

      server {
              listen 6443;
              proxy_pass api;
          }
      }

- Restart NGINX
- Create a Local DNS entry for NGINX. I used my router to create a static DNS `A` type entry that points ‘k3s.lab.devtardis.com’ to my NGINX IP which is 192.168.1.150
> Note: If using local DNS, make sure all other machines uses same local DNS such as 192.168.1.1 not public DNS as our ‘k3s.lab.devtardis.com’ is pointed to local LAN address. If you are on Cloud then you can use public DNS and have your DNS provider points to your machines public IP addresses.

---

## Install k3s

k3s is easily installed with a single command.

> We are using version: v1.23.8+k3s2 for full support of this cluster.

For our setup we will have 3 nodes and all 3 will function as both server and worker nodes which makes them HA with k3s embedded etcd.

### Install on your first node

    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.23.8+k3s2 sh -s - server --tls-san k3s.lab.devtardis.com --write-kubeconfig-mode 644 --disable servicelb --disable traefik --token "my_token" --cluster-init

> Note: Change the token value `my_token` according to your needs and this has be to a secert.

### Install on the rest of your nodes

    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.23.8+k3s2 K3S_TOKEN="my_token" sh -s - server --server https://k3s.lab.devtardis.com:6443 --write-kubeconfig-mode 644 --disable servicelb --disable traefik

Once this has been run on the rest of the nodes then you should be able to check that they are all seen by the cluster and that they are all ready to perform their duties with `kubectl get nodes`.

> If something is wrong uninstall the k3s using following.

    /usr/local/bin/k3s-uninstall.schema

### Setup kubectl on local machine

Now that we’ve seen that we’re able to interact with our cluster with `kubectl`, we’d like to be able to do that from our actual devleopment machine. Luckily, it’s fairly easy to get this configured. Install `kubectl` via this guide [https://kubernetes.io/docs/tasks/tools/]

Once kubectl is installed you’ll need to copy the contents from `/etc/rancher/k3s/k3s.yaml`, generally located on one of your nodes to `~/.kube/config` on your local machine.

> In your `config` file replace the `server:` option with your NGINX Load Balancer DNS entry we used https://k3s.lab.devtardis.com:6443

    - cluster:
        certificate-authority-data: LS-tLS1CRUdJT
        server: https://k3s.lab.devtardis.com:6443 # << change this
      name: default
    contexts:
    ...

If done correctly then you’ll be able to interact with your Kubernetes cluster from your local machine via `kubectl get nodes`.

---

## Helm

Helm is a package manager used by Kubernetes to make installing applications much easier. Follow the https://helm.sh/docs/intro/install/ directly from the Helm website on your local machine.

---

## Cert-manager

Cert-manager will be used to provide proper SSL certificates to our cluster. This will run in the background and should not require any configuration after the initial setup.

With Helm installed we can use it to easily install cert-manager. First, we will add the repo and update:

    helm repo add jetstack https://charts.jetstack.io
    helm repo update

Next, install the CRDs from the cert-manager github

> Make sure the version you use is compatible with the version of k3s you’re running we are using v1.7.1 of cert-manager.

    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

    helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.7.1 --set startupapicheck.timeout=5m

---

## MetalLB

MetalLB is a load balancer that we'll use to expose the services that our running inside of our cluster to the outside of the cluster this is seperate from our previous NGINX LB that is used for our k3s server communication. MetalLB is used as external LAN IP addresses for exposed services such as 192.168.1.165 (see below) points to mysite.com

First create a namespace.

    kubectl create namespace metallb-system

Similar to cert-manager, we will be using Helm to install MetalLB.

    helm repo add metallb https://metallb.github.io/metallb
    helm repo update
    helm install metallb metallb/metallb --namespace metallb-system

> As of v0.13, MetalLB no longer uses a ConfigMap and instead uses CRs. If you installed using Helm then chances are you're on a version that requires CRs.

Modify the [`/metallb/metallb-cr.yaml`](/metallb/metallb-cr.yaml) file to replace your DHCP IP range for MetalLB to use to expose services to your LAN with local IP or your Cloud public IPs. We are allocating total 10 IPs in this setup.

    spec:
      addresses:
      - 192.168.1.160-192.168.1.169 # << change this range.
    ---
    apiVersion: metallb.io/v1beta1

Apply your [`/metallb/metallb-cr.yaml`](/metallb/metallb-cr.yaml) file

    kubectl apply -f metallb/metallb-cr.yaml

MetalLB should now be running and ready to expose your services using the local DHCP or Cloud IP range provided.

---

## Rancher

Rancher is going to be our orchestrator for our cluster that will make monitoring, interacting, and configuring our cluster MUCH easier. Rancher will provide a UI that will be exposed using our MetalLB load balancer and accessible through any web browser.

### Install Rancher

Add Helm repo for stable rancher branch.

    helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
    helm repo update

Create a namespace to compartmentalize all of our Rancher pods.

    kubectl create namespace cattle-system

Next, install Rancher with Helm

    helm install rancher rancher-stable/rancher \
      --namespace cattle-system \
      --set hostname=rancher.lab.devtardis.com \
      --set replicas=3

> `hostname` can be either a local DNS entry or a public domain. I used my router to create a static DNS `A` type entry that points `rancher.lab.devtardis.com’ to my 192.168.1.160 which comes under MetalLB range from above.

> And `replicas` will determine how many instances should be running at a given time.

Run the following command to let you know when the Rancher rollout is complete, it will take few mins to complete.

    kubectl -n cattle-system rollout status deploy/rancher

Once Rancher is up and running you’ll want to expose the deployment on port 443 using MetalLB

    kubectl expose deployment rancher -n cattle-system --type=LoadBalancer --name=rancher-lb --port=443

> `--type=LoadBalancer` is what tells MetalLB _‘hey, take this service and expose it with one of the IP addresses in the given range’_.

To check and see if MetalLB has done it’s job, let’s take a look at our services that our now running in our cluster.

    kubectl get svc --all-namespaces -o wide

Navigating to the IP/Hostname assigned by MetalLB should display the Rancher UI and setup the password via bootstrap method given on UI at initial load.

---

## Traefik

[Traefik](https://traefik.io/) is a powerful network stack that provides many comprehensive features, however, we will primarily be using it for Ingress and as a Reverse-proxy.

### Prerequisits

**CloudFlare Domain**

- CloudFlare Domain - Move your domain over to CloudFlare's.
- Get your CloudFlare API Token
- Create a DNS entry for your site

> Note: As of now CloudFlare does not provide Glue records support for Free tier accounts. So if you need something like NS1.devtardis.com, NS2.mysite.com its only available on Business plan. It could be important for you later in the line for mail hosting, custom DNS and etc.

**Persistent Volume Claim Rancher UI Steps V2.6.x**

- From Rancher UI goto your cluster
- Click on the Storage on left side navigation and goto PersistentVolumeClaims
- Make sure you are on all namespaces on the top left of UI.
- Click on `Create`
- Choose `kube-system` as Namespace
- Name as `acme-json-certs`
- Change `Request Storage` to `1` GiB
- Leave everything default.
- Hit Create at bottom

Again, we are using Helm to install our application. But this time we will be installing with custom configurations.

First, let's add and update the repo

    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update

Modify the [`traefik/traefik-config.yaml`](/traefik/traefik-config.yaml) file that will be used to create our Cloudflare `secret` and our Traefik `ConfigMap`

> Replace `email` and `apiKey` with your Cloudflare email address and your Cloudflare API Token (Not Global Token)

    type: Opaque
    stringData:
    email: xxxxxxxxxxxx@gmail.com # change this.
    apiKey: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx # change this
    ---

Modify the values in [`/traefik/traefik-chart-values.yaml`](/traefik/traefik-chart-values.yaml) file that will provide custom values when we install Traefik with Helm

    --providers.file.filename=/data/traefik-config.yaml
    --entrypoints.websecure.http.tls.certresolver=cloudflare
    --entrypoints.websecure.http.tls.domains[0].main=lab.devtardis.com # change this
    --entrypoints.websecure.http.tls.domains[0].sans=*.lab.devtardis.com # change this

...

    spec: # externalTrafficPolicy: Cluster
    loadBalancerIP: "192.168.88.161" # change this the IP you would like assigned via Metallb

> Replace `--enterypoints` options with your Domain / Subdomain and adjust the `loadBalancerIP:` to the IP you’d like MetalLB to assign to Traefik (must be in the range you specified earlier)

Now, lets apply the config, which will create our secret and ConfigMap

    kubectl apply -f /traefik/traefik-chart-values.yaml

Install Traefik with Helm using our charts YAML

    helm install traefik traefik/traefik --namespace=kube-system --values=/traefik/traefik-chart-values.yaml

> If something is wrong uninstall the traefik using following.

    helm uninstall traefik traefik/traefik --namespace=kube-system

Check if Traefik is running properly

    kubectl -n kube-system logs $(kubectl -n kube-system get pods --selector "app.kubernetes.io/name=traefik" --output=name)

Should return something like this:

    level=info msg="Configuration loaded from flags."

### Expose Traefik Dashboard

You'll probably want to be able to access your new Traefik UI, so let's get that done.

Modify the [`/traefik/traefik-dashboard-auth.yaml`](/traefik/traefik-dashboard-auth.yaml) file which will contain a base64 string secret used to initially log into our dashboard.

    users: dXNlcjokYXByMSR2eG9yTDY2WCRTNllubDU5cUNuVXE3UWZYSmNtMGcvCgo= # cahnge this.

Use the following on local machine to generate new base64 string for your login.

    htpasswd -nb user password | openssl base64

Modify the [`/traefik/traefik-dashboard-ingressroute.yaml`](/traefik/traefik-dashboard-ingressroute.yaml) file to expose the dashboard itself and change the Hostname DNS entry.

    routes:
    - match: Host(`traefik.lab.devtardis.com`) # change this
      kind: Rule

> Make sure that DNS is pointed to the IP assigned by MetalLB.

Now we can go ahead and apply these configs

    kubectl apply -f traefik-dashboard-auth.yaml
    kubectl apply -f traefik-dashboard-ingressroute.yaml

If done correctly you should now be able to access your Traefik dashboard using the URL specified in your ingress route config.

Now you’re setup with a HA k3s cluster with a MetalLB load balancer, and Traefik ingress. From here you can properly expose Services to the outside world and they'll have proper TLS/SSL with cert-manager and CloudFlare.

---
Last Update:
1 Oct 2022