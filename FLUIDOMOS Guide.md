# FLUIDOMOS setup and installation guide

This guide shows the steps required to implement the FLUIDOMOS project infrastructure.
The project test case creates a home automation system installed at the Vemar SAS company managed through the DELIS platform running on a K3S multi-cluster environment.
In detail, the infrastructure includes two clusters with the roles of Consumer and Provider, equipped with FLUIDOS technology.

The Consumer cluster will manage the Edge nodes delegated to interface with the home automation devices. Specifically, a single worker node was added and configured on Edge hardware (Raspberry Pi 4) delegated to interface with all the home automation devices installed at Vemar.

The Provider cluster, with a single worker node, based on a Virtual Private Server, will offer resources to the Consumer cluster.

A table with further details of the infrastructure is shown below

|        IP        |Name                            |Role/Resource                |
|------------------|--------------------------------|-----------------------------|
|91.121.49.150 10.255.0.131|**consumer1-fluidomos**|**Consumer** cluster control plane|
|||*OS: Debian 12.6 – RAM: 32GB – 8 CPU – Disk: 20GB – Virtual machine amd64*
|10.255.0.132         |**cworker1-fluidomos** |Worker VPS-node of Consumer cluster|
|||*OS: Debian 12.6 – RAM: 32GB – 8 CPU – Disk: 20GB – Virtual machine amd64*|
|xxx.xxx.xxx.xxx        |**cedge1-fluidomos** |Worker Edge-node of Consumer cluster*|
|||*Quad core Cortex-A72 (ARM v8) 64-bit SoC @ 1.8GHz 8GB 2.4 GHz– Raspberri PI 4*|
|10.255.36.163|**cprovider1-fluidomos**|**Provider** cluster control plane|
|||*OS: Debian 12.6 – RAM: 32GB – 8 CPU – Disk: 20GB – Virtual machine amd64*
|110.225.36.164        |**pworker1-fluidomos** |Worker VPS-node of Provider cluster |
|||*OS: Debian 12.6 – RAM: 32GB – 8 CPU – Disk: 20GB – Virtual machine amd64*|

The basic products (helm, k3s, liqo) were installed on the master nodes ( **consumer1-fluidomos** and **provider1-fluidomos** ) and exclusively k3s on the worker nodes ( **cworker1-fluidomos**, **cedge1-fluidomos** and **pworker1-fluidomos** ) in the sequences indicated below.


## Steps

###  1 Installing Helm on both master nodes consumer1-fluidomos and provider1-fluidomos

```bash
apt-get update && apt-get install apt-transport-https gpg git curl unzip

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | tee /usr/share/keyrings/helm.gpg > /dev/null

echo "deb \[arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg\] https://baltocdn.com/helm/stable/debian/ all main" | tee /etc/apt/sources.list.d/helm-stable-debian.list

apt get update

apt-get install helm
```

### 2 Installing K3s to version v1.24.17+k3s1 on all control-plane and worker nodes

#### 2.1 Setup the consumer1-fluidomos
```bash
curl -sfL https://get.k3s.io | K3S\_NODE\_NAME=consumer1-fluidomos INSTALL\_K3S\_VERSION=v1.24.17+k3s1 sh -s - server --cluster-init
```
to get the token type (please see your own generated token) :
```bash
cat /var/lib/rancher/k3s/server/node-token  
```
result 
```bash
K10671656dd98b4604a78fbea80ccbd5a7ca4695fe2372b70b98bb10dcOBSCURED::server:dc37106222de489c00e4457d73e9OBSCURED
```
The control plane token must be used for adding nodes to the Consumer cluster

#### 2.2 Setup cworker1-fluidomos
```bash
curl -sfL https://get.k3s.io | INSTALL\_K3S\_EXEC="agent" K3S\_URL=https://10.255.0.131:6443 K3S\_NODE\_NAME=cworker1-fluidomos INSTALL\_K3S\_VERSION=v1.24.17+k3s1 K3S\_TOKEN=K10671656dd98b4604a78fbea80ccbd5a7ca4695fe2372b70b98bb10dcOBSCURED::server:dc37106222de489c00e4457d73e9OBSCURED sh -
```
#### 2.3 Setup **cedge1-fluidomos**
Modify the file 
```bash
cat /boot/firmware/cmdline.txt
```
Add the values  for **cgroup_memory** and **cgroup_enable** as follow:
```bash
console=serial0,115200 console=tty1 root=PARTUUID=f83099fb-02 rootfstype=ext4 fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory
```
then exec 
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" K3S_URL=https://91.121.49.150:6443 K3S_NODE_NAME=cedge1-fluidomos INSTALL_K3S_VERSION=v1.24.17+k3s1 K3S_TOKEN=K10671656dd98b4604a78fbea80ccbd5a7ca4695fe2372b70b98bb10dcOBSCURED::server:dc37106222de489c00e4457d73e9OBSCURED sh -
```

#### 2.4 Setting up  provider1-fluidomos
```bash
curl -sfL https://get.k3s.io | K3S\_NODE\_NAME=provider1-fluidomos INSTALL\_K3S\_VERSION=v1.24.17+k3s1 sh -s - server –cluster-init
```
to get the token type (please see your own generated token) :
```bash 
cat /var/lib/rancher/k3s/server/node-token  
```
result
```bash 
K102d9ec76ba1f35bbc9c763739caa01fb6b78b25f7f3af42cbfa57e9aadOBSCURED::server:b2c73c0e85cc48a8dafe8f8a2262OBSCURED
```
The control plane token must be used for adding nodes to the Provider cluster

#### 2.5 Setting up  pworker1-fluidomos
```bash
curl -sfL https://get.k3s.io | INSTALL\_K3S\_EXEC="agent" K3S\_URL=https://10.255.36.163:6443 K3S\_NODE\_NAME=pworker1-fluidomos INSTALL\_K3S\_VERSION=v1.24.17+k3s1 K3S\_TOKEN=K102d9ec76ba1f35bbc9c763739caa01fb6b78b25f7f3af42cbfa57e9aadOBSCURED::server:b2c73c0e85cc48a8dafe8f8a2262OBSCURED sh -
```
### 3 Installing Liqoctl CLI 
On both **consumer1-fluidomos** and **provider1-fluidomos** type:

```bash
cd /tmp

curl --fail -LS "https://github.com/liqotech/liqo/releases/download/v0.10.3/liqoctl-linux-amd64.tar.gz" | tar -xz

install -o root -g root -m 0755 liqoctl /usr/local/bin/liqoctl

rm liquoctl
```
### 4 FLUIDOS Setup

#### 4.1 Copying FLUIDOS package v.0.0.5 

On both master nodes **consumer1-fluidomos** and **provider1-fluidomos**
```bash
cd /root  
wget [https://github.com/fluidos-project/node/archive/refs/tags/v0.0.5.zip](https://github.com/fluidos-project/node/archive/refs/tags/v0.0.5.zip)  
unzip v0.0.5.zip  
mv node-0.0.5 node
```
#### 4.2 Label all node of Consumer cluster 

On **consumer1-fluidomos**
```bash
kubectl label nodes consumer1-fluidomos node-role.fluidos.eu/resources=true node-role.fluidos.eu/worker=true  
```
Om **cworker1-fluidomos**
```bash
kubectl label nodes cworker1-fluidomos node-role.fluidos.eu/resources=true node-role.fluidos .eu/worker=true
```
#### 4.3 Label all node of Provider cluster 
On **provider1-fluidomos**
```bash
kubectl label nodes provider1-fluidomos node-role.fluidos.eu/resources=true node-role.fluidos.eu/worker=true  
```
On **pworker1-fluidomos**
```bash
kubectl label nodes pworker1-fluidomos node-role.fluidos.eu/resources=true node-role.fluidos .eu/worker=true
```
#### 4.4 Installing YAML and adding FLUIDOS repository

The following commands are executed on **consumer1-fluidomos** and **provider1-fluidomos**
```bash
cd node/testbed/kind

kubectl apply -f ../../deployments/node/crds

export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

helm repo add fluidos https://fluidos-project.github.io/node/
```
#### 4.5 Installing  FLUIDOS node and LIQOCTL on consumer1-fluidomos 

```bash
cd /root/node/testbed/kind
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

helm install node fluidos/node -n fluidos 
--create-namespace -f consumer/values.yaml 
--set networkManager.configMaps.nodeIdentity.ip="91.121.49.150:30000" 
--set networkManager.configMaps.providers.local="10.255.36.163:30001" 
--version 0.0.5

liqoctl install k3s --cluster-name fluidos-consumer 
--version v0.10.3
--set controllerManager.config.resourcePluginAddress=node-rear-controller-grpc.fluidos:2710
--set controllerManager.config.enableResourceEnforcement=true
```
> Note: The cluster-name fluidos-consumer cannot be changed

#### 4.6 Installing  FLUIDOS node and LIQOCTL on provider1-fluidomos 

```bash
cd /root/node/testbed/kind

export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

helm install node fluids/node -n fluids
--create-namespace -f provider/values.yaml \\
--set networkManager.configMaps.nodeIdentity.ip="10.255.36.163:30001" 
--set networkManager.configMaps.providers.local="91.121.49.150:30000" 
--version 0.0.5

liqoctl install k3s --cluster-name fluids-provider \\
--version v0.10.3 \\
--set controllerManager.config.resourcePluginAddress=node-rear-controller-grpc.fluidos:2710 \\
--set controllerManager.config.enableResourceEnforcement=true
```

#### 4.7 Check the installation status of fluidos and liquo
To check installation status use the following coomands:

```bash
kubectl get pods -A
liquoctl status
kubectl get flavors.nodecore.fluidos.eu -n fluidos
```
### 5 Using  FLUIDOS 

Before acting the FLUIDOS peering to consume Provider resources, run the following commands on **consumer1-fluidomos** to set the FLUIDOS solver
```bash
nano node/deployments/node/samples/solver.yaml
```
Change the following parameters we estabilish for the FLUIDOMOS project:
```yaml
rangeSelector:
  minCpu: "6000m"
  minMemory: "24Gi"
  minPods: "100"
```
Finally to estabilish the peering type:
```bash
kubectl apply -f node/deployments/node/samples/solver.yaml
```
To check:
```bash
kubectl get solver -n fluidos
```
### 6 Check FLUIDOS peering
Finally a one-way peering from consumer to provider is established
To check  on consumer type:
```bash 
root@consumer:~# export KUBECONFIG=/etc/rancher/k3s/k3s.yaml 
liqoctl status peer

┌─ Peered Cluster Information ───────────────────────────┐
| fluidos-provider- 32e707ce-70eb-49af-a288-74c079f91e0e |
| Type: OutOfBand                                        |
| Direction                                              |
|    Outgoing: Established                               |
|    Incoming: None                                      | 
| Authentication                                         |
|    Status: Established                                 |
| Network                                                |
|    Status: Established                                 |
| Network connection                                     |
|    Status: Connected                                   |
| Latency: 25ms                                          | 
| IP Gateways                                            |
|    Local: 91.121.49.150:32645                          |
|    Remote: 10.255.36.163:32767                         |
| API Server                                             |
|   Status: Established                                  |
| Resources                                              |
| Total acquired - resources offered                     |
|                        by "fluidos-provider"           |
|                        to "fluidos-consumer"           |
|   cpu: 6000m                                           |
|   memory: 25.17GiB                                     |
|   pods: 100                                            |
|   ephemeral-storage: 0.00GiB                           |
└────────────────────────────────────────────────────────┘
```
and on Provider

```bash 
root@consumer:~# export KUBECONFIG=/etc/rancher/k3s/k3s.yaml 
liqoctl status peer

┌─ Peered Cluster Information ───────────────────────────┐
| fluidos-provider-f0403bd6-be56-425a-a8d4-03eebdad982b  |
| Type: OutOfBand                                        |
| Direction                                              |
|    Outgoing: None                                      |
|    Incoming: Established                               | 
| Authentication                                         |
|    Status: Established                                 |
| Network                                                |
|    Status: Established                                 |
| Network connection                                     |
|    Status: Connected                                   |
| Latency: 24ms                                          | 
| IP Gateways                                            |
|    Local: 10.255.36.163:32767                          |
|    Remote: 91.121.49.150:32645                         |
| API Server                                             |
|   Status: Established                                  |
| Resources                                              |
| Total acquired - resources offered                     |
|                        by "fluidos-provider"           |
|                        to "fluidos-consumer"           |
|   cpu: 6000m                                           |
|   memory: 25.17GiB                                     |
|   pods: 100                                            |
|   ephemeral-storage: 0.00GiB                           |
└────────────────────────────────────────────────────────┘
```
### 7 FLUIDOMOS namespaces Configuration and Setup

On consumer we create two namaspace for the FLUIDOMOS project. The namespace definition are in the following YAML file located in the project repository:

**namespace-fluidomos-datacenter.yam**
```yaml
apiVersion: v1  
kind: Namespace  
metadata:
  name: fluidomos-datacenter
  labels:
    name: fluidomos
    layer: orchestration  
```
**namespace-fluidomos-edge.yaml**  
```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: fluidomos-edge
  labels:
    name: fluidomos
    layer: orchestration
```
To create the namespace type:
```bash 
root@consumer1-fluidomos:~/delis# kubectl apply -f namespace-fluidomos-datacenter.yaml
namespace/fluidomos-datacenter created

root@consumer1-fluidomos:~/delis# kubectl apply -f namespace-fluidomos-edge.yaml
namespace/fluidomos-edge created
```
### 8 FLUIDOMOS K3s Secrets
The K3s Secrets to access the private Vemar registry, containing the docker images of the Delis project, can be created with the following commands:

```bash 
root@consumer1-fluidomos:~/delis# kubectl create secret docker-registry regcred --namespace fluidomos-datacenter --docker-server=registry.vemarsas.it --docker-username=USER --docker-password=PASSWORD --docker-email=monitoraggio@vemarsas.it
secret/regcred created

root@consumer1-fluidomos:~/delis# kubectl create secret docker-registry regcred --namespace fluidomos-edge --docker-server=registry.vemarsas.it --docker-username= USER --docker-password=PASSWORD --docker-email=monitoraggio@vemarsas.it 
secret/regcred created
```
> To obatin the USER and PASSWORD values contact the FLUIDOMOS Project Manager.

### 9 Delis Deployment
To install DELIS apply the following deployment in the consumer. All the YAML file are located in the project repository

 - deployment_cable.yaml
 - deployment_pydelis_zway.yaml
 - deployment_sidekiq.yaml
 - deployment_wc.yaml
 - service_cable.yaml
 - service_pydelis_zway.yaml
 - service_sidekiq.yaml
 - service_wc.yaml
 - 
### 10 Offloading
To using FLUIDOS offload, on the consumer typ ;
```bash
root@consumer1-fluidomos:~/delis# export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"
root@consumer1-fluidomos:~/delis# liqoctl offload namespace fluidomos-datacenter

INFO Offloading of namespace "fluidomos-datacenter" correctly enabled  
INFO Offloading completed successfully

root@consumer1-fluidomos:~/delis# liqoctl offload namespace fluidomos-edge

INFO Offloading of namespace "fluidomos-edge" correctly enabled  
INFO Offloading completed successfully
```
To show the results type:
```bash
root@consumer1-fluidomos:~/delis# kubectl get pods -n fluidomos-datacenter -o wide

NAME                   READY STATUS  RESTARTS AGE IP          NODE                  NOMINATED NODE READINESS GATES
------------------------------------------------------------------------------------------------------------------
cable-6b776c9f48-bhxhm  1/1  Running 0       31s 10.42.2.194 cworker1-fluidomos     <none>        <none>
cable-6b776c9f48-btwqd  1/1  Running 0       31s 10.41.0.109 liqo-fluidos-provider  <none>        <none>
cable-6b776c9f48-ds4j4  1/1  Running 0       31s 10.41.1.106 liqo-fluidos-provider  <none>        <none>
cable-6b776c9f48-qqt9s  1/1  Running 0       31s 10.42.2.193 cworker1-fluidomos     <none>        <none>
sidekiq-f9fb67fcc-8l8fj 1/1  Running 0       31s 10.42.2.191 cworker1-fluidomos     <none>        <none>
sidekiq-f9fb67fcc-gl6hh 1/1  Running 0       31s 10.42.0.199 consumer1-fluidomos    <none>        <none>
sidekiq-f9fb67fcc-qpzdv 1/1  Running 0       31s 10.42.0.198 consumer1-fluidomos    <none>        <none>
sidekiq-f9fb67fcc-t296h 1/1  Running 0       31s 10.42.2.192 cworker1-fluidomos     <none>        <none>
wc-6f499f4786-nnjft     1/1  Running 0       31s 10.42.0.200 consumer1-fluidomos    <none>        <none>
wc-6f499f4786-rr472     1/1  Running 0       31s 10.42.2.195 cworker1-fluidomos     <none>        <none>
```
```bash
root@consumer1-fluidomos:~/delis# kubectl get pods -n fluidomos-edge -o wide

NAME                         READY STATUS RESTARTS AGE IP          NODE NOMINATED NODE READINESS GATES
------------------------------------------------------------------------------------------------------
pydelis-zway-6c5997894f-77mvz 1/1  Running 0       37s 10.42.3.35  cedge1-fluidomos      <none> <none>
pydelis-zway-6c5997894f-78r7s 1/1  Running 0       37s 10.42.2.189 cworker1-fluidomos    <none> <none>
pydelis-zway-6c5997894f-prqrj 1/1  Running 0       37s 10.42.2.190 cworker1-fluidomos    <none> <none>
pydelis-zway-6c5997894f-ssg6j 1/1  Running 0       37s 10.41.1.104 liqo-fluidos-provider <none> <none>
pydelis-zway-6c5997894f-tdmkr 1/1  Running 0       37s 10.41.1.103 liqo-fluidos-provider <none> <none>
pydelis-zway-6c5997894f-xq9c2 1/1  Running 0       37s 10.41.1.105 liqo-fluidos-provider <none> <none>
