# Build up K8s cluster for general AI/ML development

This document is a general guidance about how to prepare a development/cooperation environment for a small AI/ML team with bare local servers with CUDA devices.

## Hardware description

Generally, the hardware consists of several baremetal workstations (with CUDA devices equipped), and a Synology NAS station server. The workstations and NAS server are connected via dumb switch w/o management function.

## Preparation

Before establish the environment, here we need to do some preparation.

### DHCP server and DNS server

We need an easy networking environment where each workstation and NAS can access each other without any obstales. Thus we should make sure the internal networking is configured as follows:

- The internal network has a internal or external domain name. e.g. internal.example.com
- Every workstation can access each other via a fixed IP address as well as a hostname/FQDN

To achieve the upper 2 requirement, we need a proper configured DHCP/DNS server which have Mac address/IP binding and internal FQDN resolution. Normally there are 2 options.

- Option 1. Use router to provide DNS service and DHCP service. This requires a router which have IP reservation and DNS name resolution capability.

- Option 2. User Synology NAS to provide DHCP and DNS solution. On Synology NAS, we can install following 2 packages to provide DHCP and DNS function. For detailed info, we can refer the Synology document.

### User Authentication management

There needs to be a directory service which could provide single point of user authentication. We can choose either of following 2 options.

- Option 1. LDAP service on Synology NAS. Please check [LDAP Server](https://www.synology.com/en-global/knowledgebase/DSM/help/DirectoryServer/ldap_desc)

- Option 2. Authentication service running on cloud. E.g. [Okta](https://www.okta.com/products/single-sign-on/).

### Firewall setting

We should limit the accessibility towards internal network, this can be done via router's configuration for small business.

## Cluster infrastructure establishment

Here are the steps about how to create a Kubernetes cluster on top of the baremetal workstations.

### Operating system

We choose Ubuntu 20.04 server Focal LTS as the base operating system since it is easy to be managed and have the best community support so far. We should disable "root" user but create a unified admin account on every machine and forbid the password authentication.

### Necessary software installation

Following software are crucial for composing a Kubernetes cluster and make use of the CUDA hardware. Thus, you need to install all the software mentioned below on every single workstation.

- openssh-server

  ```bash
  sudo apt-get update && sudo apt-get install openssh-server -y
  ```

- [docker](https://docs.docker.com/engine/install/ubuntu/)
  
  ```bash
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

  sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

  sudo apt-get update && sudo apt-get install 
  ```

- [nvidia-docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)
  
  ```bash
  # nvidia-docker is a 3pp add-on for docker
  distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

  sudo apt-get update && sudo apt-get install nvidia-docker2

  ```

  Docker configuration needs to be updated in order to make Nvidia environment become effective. Open "/etc/docker/daemon.json" file and make sure it looks like below. (Create it if the file doesn't exist)

  ```json
  {
    "storage-driver": "overlay2",
    "default-runtime": "nvidia",
    "features": {
    "buildkit": true
    },
    "runtimes": {
      "nvidia": {
        "path": "nvidia-container-runtime",
        "runtimeArgs": []
      }
    }
  }
  ```

### Install Kubernetes via RKE

We'll use a simple program named [RKE](https://rancher.com/products/rke/) to deploy entire Kubernetes infrastructure. Here are the general steps.

- Step 1. Install RKE on a laptop connected to the same LAN with the . Please follow [RKE Installation Guide](https://rancher.com/docs/rke/latest/en/installation/) to install RKE binary on the laptop.

- Step 2. Preapare **cluter.yaml** file with following content, remember to replate the address and user with correct value.

  ```yaml
  nodes:
  - address: <FQDN of 1st workstation computer>
    user: <Admin User>
    role:
      - controlplane
      - etcd
      - worker
  - address: <FQDN of 2nd workstation computer>
    user: <Admin User>
    role:
      - controlplane
      - etcd
      - worker
  - address: <FQDN of 3rd workstation computer>
    user: <Admin User>
    role:
      - etcd
      - worker
  # Please add more nodes here according to hardware status
  ...

  # If set to true, RKE will not fail when unsupported Docker version
  # # are found
  ignore_docker_version: true
  ```

- Step 3. Install the Kubernetes with following command.
  
  ```bash
  rke up
  ```

- Step 4. Copy the Kubernetes config file to default location

  ```bash
  mkdir -p ~/.kube
  cp kube_config_cluster.yml ~/.kube/config
  ```

### Use Helm to deploy the necessary services

We need to deploy necessary services into the vanila Kubernetes cluster to support our AL/ML development applications.

#### Install Helm on your laptop

In order to use Helm, we should install it on our client laptop following [Helm installation guide](https://helm.sh/docs/intro/install/).

#### Deploy MetalLB

[Metallb](https://metallb.universe.tf/) is a simple tool which provide LoadBalancer capability to the K8s cluster and provide single entry IP for multiple services. This simplifies the external routing. Remember to replace the **start-ip-address** and **end-ip-address** to the address in you local LAN. E.g. if your local LAN is 192.168.11.0/24, you can pick 192.168.11.100 as start-ip-address and 192.168.11.120 as end-ip-address. N.B. Make sure you DHCP server won't assign IP address within this range to avoid potential collision.

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
kubectl create namespace metallb
helm install metallb stable/metallb --namespace metallb \
  --set configInline.address-pools[0].name=default \
  --set configInline.address-pools[0].protocol=layer2 \
  --set configInline.address-pools[0].addresses[0]=<start-ip-address>-<end-ip-address>
```

#### Deploy cert-manager

[Cert-Manager](https://cert-manager.io/) is the tool to help us get/store/manage TLS certificate from [Letsencrypt](https://letsencrypt.org/) easily.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0 \
  --create-namespace \
  -set installCRDs=true
```

#### Deploy nginx-ingress-controller

[Nginx-Ingress-Controller](https://www.nginx.com/products/nginx-ingress-controller/) is the tool helping us to route http/https traffic to correct service.

```bash
kubectl create namespace ingress-controller
helm install nginx-ingress stable/nginx-ingress \
  --namespace ingress-controller
```

#### Deploy Rancher

[Rancher](https://rancher.com/products/rancher/) is a very nice tool to visualise and manage the cluster. The reference doc is [Install Rancher on Kubernetes Cluster](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/helm-rancher/). Remember to change **EMAIL_ADDR** to a valid email address

```bash
kubectl create namespace cattle-system
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=<HOST_NAME> \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=<EMAIL_ADDR>
```

After Rancher is installed, we could check if Rancher is accessible from WAN and LAN.

![rancher-login](https://user-images.githubusercontent.com/944672/81685208-2f4f9900-9458-11ea-8e03-6aab82e399b9.png)

The cluster is shown in below pic.

![K3s Local Cluster](https://user-images.githubusercontent.com/944672/82371075-e0869e00-9a19-11ea-8ecd-0c8516faaa56.png)

#### Prepare cluster-issuer for further deployment

In order to make use of cert-managet to other services, create cluster-issuer as below. Remember to change **EMAIL** to a valid email address.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: <EMAIL>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```
