# Introduction
This instruction aims to describe how to install Rancher Manager on WSL2, by using K3d instance, that is capable of adding downstream clusters on another K3d instance.

# Prerequisites
WSL2 installed with Ubuntu 22.04 instance installed.
Docker installed on Ubuntu - steps from official documentation:

* https://docs.docker.com/engine/install/ubuntu/ 
* https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user

# Install needed CLIs
Tip. Homebrew can install all CLIs, but it is also possible to install each with a script, curl, or similar.

Autocomplete is shown for bash, but in most cases, source pages show steps for zsh and fish.
## K3d
### Install
Please follow the guidelines from these pages:

* https://github.com/k3d-io/k3d
or
* https://k3d.io/v5.6.3/

Check version
```bash
k3d version
```
In a result, you should get version (or higher)
```bash
k3d version v5.6.0
k3s version v1.27.4-k3s1 (default)
```
### Autocomplete

Please execute 
```bash
k3d completion bash > /usr/local/etc/bash_completion.d/k3d
```
For other shell types please visit:
* https://k3d.io/v5.0.1/usage/commands/k3d_completion/

## kubectl 
### Install 

Please follow the steps from this page (pick any method):
* https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

Check if works
In response, you should get 
```
Client Version: v1.29.2
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```
In practice, you need to have a version equal, to or higher, than Kubernetes'. 
### Autocomplete
Please type 
```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
For other shells please visit 
* https://kubernetes.io/docs/reference/kubectl/quick-reference/#kubectl-autocomplete

## helm
### Install 
Please follow the steps from this page:
* https://helm.sh/docs/intro/install/

Please check if it is installed successfully:
```bash
helm version
```

In a result, you should get:
```bash
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}
```
### Autocomplete
Please type:
```bash
source <(helm completion bash)
helm completion bash > /etc/bash_completion.d/helm
```

For other shells please check:
* https://helm.sh/docs/helm/helm_completion/

# Install Rancher Manager
## Create a cluster in K3d
1. Please create a cluster named 'rancher-manager'
```bash
k3d cluster create rancher-manager --network host
```
2. Check if you can successfully connect
```bash
kubectl cluster-info
```
In response you should get:
```bash
Kubernetes control plane is running at https://0.0.0.0:6443
CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
3. Check the Docker container list
```bash
docker ps
```
In response, you should get 
```
CONTAINER ID   IMAGE                            COMMAND                  CREATED        STATUS        PORTS                             NAMES
702a89a127f3   rancher/k3s:v1.27.4-k3s1         "/bin/k3s server --t…"   24 hours ago   Up 24 hours                                     k3d-rancher-manager-0
```
There should be no load balancer containers.
## Install Cert-Manager
1. Install CRDs
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
```
2. Add helm repository
```bash
helm repo add cert-manager https://charts.jetstack.io
helm repo update
```
3. Install Cert-Manager in `cert-manager` namespace
```bash
helm -n cert-manager install cert-manager cert-manager/cert-manager --version 1.14.5 --create-namespace
```
Check if the installation was successful
```bash
kubectl -n cert-manager rollout status deployment cert-manager
kubectl -n cert-manager get pods
helm list -n cert-manager
```
## Install Rancher Manager with a self-signed certificate
1. Add helm repository
```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```
2. Check your IP
```bash
ip a | grep eth0: -A3 | grep inet
```
3. Install Rancher
```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.<IP>.sslip.io \
  --set bootstrapPassword=superSecretPassword
  --create-namespace
```
4. Check if Rancher was installed successfully
```bash
kubectl -n cattle-system rollout status
kubectl -n cattle-system get ingress
```
5. Open Rancher Manager in a browser `https://rancher.<IP>.sslip.io`
6. Type the password set in Step 3
7. Check the license agreement

## Join a new cluster to the Rancher Manager
1. Create a new cluster `downstream`
```bash
k3d cluster create downstream
k3d cluster list
kubectl cluster-info # after creation kubeconfig context should be changed to a new cluster
```
2. In Rancher Manager import an existing cluster. Open `Cluster Management` > `Import Existing` > `Generic`.
Set name and click `Create`
3. Pick all roles and set insecure.
4. Copy the command and paste it into the shell.
5. In Rancher Manager check the status.

