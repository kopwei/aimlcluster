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

Following software are crucial for composing a Kubernetes cluster and make use of the CUDA hardware.

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

In order to use Helm, we should install it on our client laptop following [Helm installation ]

#### Deploy MetalLB

[Metallb](https://metallb.universe.tf/) is a simple tool which provide LoadBalancer capability to the K8s cluster and provide single entry IP for multiple services. This simplifies the external routing.

#### Deploy Rancher

[Rancher](https://rancher.com/products/rancher/) is a very nice tool to visualise and manage the cluster.
