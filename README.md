# cloudinit-k8s


To provision the lab environment we will be using cloud-init from Canonical to customize our virtual machines.

Quick start
If you’re on Linux, Mac, or have access to a Bash shell on Windows you can follow these steps to get up and running quickly:


```
curl https://raw.githubusercontent.com/tigera/ccol1/main/control-init.yaml | multipass launch -n control -m 2048M 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node1-init.yaml | multipass launch -n node1 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/node2-init.yaml | multipass launch -n node2 20.04 --cloud-init -
curl https://raw.githubusercontent.com/tigera/ccol1/main/host1-init.yaml | multipass launch -n host1 20.04 --cloud-init -
```
Windows
If you’re on Windows, in powershell you can follow these steps to get up and running:

```
Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/tigera/ccol1/main/control-init.yaml' | Select-Object -Expand Content | multipass launch -n control -m 2048M 20.04 --cloud-init -
Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/tigera/ccol1/main/node1-init.yaml' | Select-Object -Expand Content | multipass launch -n node1 20.04 --cloud-init -
Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/tigera/ccol1/main/node2-init.yaml' | Select-Object -Expand Content | multipass launch -n node2 20.04 --cloud-init -
Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/tigera/ccol1/main/host1-init.yaml' | Select-Object -Expand Content | multipass launch -n host1 20.04 --cloud-init -
```
Starting the Instances
On some platforms, multipass requires you to start the VMs after they have been launched. We can do this by using the multipass start command.

```
multipass start --all
```
Throughout the deployments for the labs, the instances will reboot once provisioning is complete. As a result, you may have to wait a minute until the instance has fully provisioned. A quick way to check the current state of the cluster is to use the multipass list command.

```
multipass list
```
Example output:
```
Name                    State             IPv4             Image
control                 Running           172.17.78.3      Ubuntu 20.04 LTS
host1                   Running           172.17.78.6      Ubuntu 20.04 LTS
node1                   Running           172.17.78.7      Ubuntu 20.04 LTS
node2                   Running           172.17.78.12     Ubuntu 20.04 LTS
```
Validating the Environment
To validate the lab has successfully started after all four instances we will enter the host1 shell:

```
multipass shell host1
```
Once you reach the command prompt of host1, run kubectl get nodes.
```
kubectl get nodes -A
```
Example output:
```
NAME      STATUS     ROLES    AGE     VERSION
node1     NotReady   <none>   4m44s   v1.18.10+k3s1
node2     NotReady   <none>   2m48s   v1.18.10+k3s1
control   NotReady   master   6m36s   v1.18.10+k3s1
```
Note the “NotReady” status. This is because we have not yet installed a CNI plugin to provide the networking.

The instance we will be using for the following labs will be host1 unless otherwise specified. Think of host1 as your primary entry point into the kubernetes ecosystem, with the other instances acting as the cluster in the cloud.

We will be using the Tigera Operator to install and configure Calico.

If you are not already on host1, you can enter host1 by using the multipass shell command.
```
multipass shell host1
```
The command below will install the operator onto our lab kubernetes cluster:
```
kubectl create -f https://docs.projectcalico.org/archive/v3.16/manifests/tigera-operator.yaml
```
Validating the Operator installation
Following the operator being installed, we will validate that the operator is running:
```
kubectl get pods -n tigera-operator
```
The output from this command should indicate that the operator pod is running:
```
NAME                               READY   STATUS    RESTARTS   AGE
tigera-operator-64f448dfb9-d2fdq   1/1     Running   0          2m33s
```
Installing Calico
After the operator is in a Running state, we will configure an Installation kind for Calico, specifying the IP Pool that we would like below.

Note that throughout the course we make use of inline manifests (piping stdin to kubectl) to make it easier for you to follow what each manifest does. In most cases it would be a more normal practice to use a vanilla kubectl command with a manifest file (e.g. kubectl apply -f my-installation.yaml).  We recommend taking a minute to read through and make sure you understand the contents of each manifest we apply in this way throughout the rest of the course to get the most out of each example.
```
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    containerIPForwarding: Enabled
    ipPools:
    - cidr: 198.19.16.0/21
      natOutgoing: Enabled
      encapsulation: None
EOF
```
Validating the Calico installation
Following the configuration of the installation resource, Calico will begin deploying onto your cluster. This can be validated by running the following command:
```
kubectl get tigerastatus/calico
```
The output from the command when the installation is complete is:

NAME     AVAILABLE   PROGRESSING   DEGRADED   SINCE
calico   True        False         False      10m
We can review the environment now by invoking:
```
kubectl get pods -A
Example output:

NAMESPACE         NAME                                      READY   STATUS      RESTARTS   AGE
tigera-operator   tigera-operator-84c5c5d6df-zb49b          1/1     Running     0          5m48s
calico-system     calico-typha-868bb997ff-l22n7             1/1     Running     0          4m6s
calico-system     calico-typha-868bb997ff-fvmws             1/1     Running     0          3m24s
calico-system     calico-typha-868bb997ff-8qt45             1/1     Running     0          3m24s
calico-system     calico-node-r94mp                         1/1     Running     0          4m6s
calico-system     calico-node-w5ptt                         1/1     Running     0          4m6s
calico-system     calico-node-zgrvb                         1/1     Running     0          4m6s
kube-system       helm-install-traefik-t68vd                0/1     Completed   0          35m
kube-system       metrics-server-7566d596c8-pccvz           1/1     Running     2          35m
kube-system       local-path-provisioner-6d59f47c7-gh97b    1/1     Running     2          35m
calico-system     calico-kube-controllers-89cf65556-c7gz7   1/1     Running     3          4m6s
kube-system       coredns-7944c66d8d-f4q6g                  1/1     Running     0          35m
kube-system       svclb-traefik-9bxg2                       2/2     Running     0          32s
kube-system       svclb-traefik-pb72f                       2/2     Running     0          32s
kube-system       svclb-traefik-l6mzn                       2/2     Running     0          32s
kube-system       traefik-758cd5fc85-8hcdx                  1/1     Running     0          32s
```
Reviewing Calico pods
Let's take a look at the Calico pods that have been installed by the operator.
```
kubectl get pods -n calico-system
Example Output:

NAME                                       READY   STATUS    RESTARTS   AGE
calico-typha-5d788c654b-56wp9              1/1     Running   0          4h28m
calico-node-2bkv5                          1/1     Running   0          4h28m
calico-kube-controllers-5dcfdbc5f4-vpgx5   1/1     Running   0          4h28m
calico-node-8465h                          1/1     Running   0          4h26m
calico-typha-5d788c654b-wn7gf              1/1     Running   0          4h24m
calico-node-qmq5j                          1/1     Running   0          3h57m
calico-typha-5d788c654b-rd8kl              1/1     Running   0          3h56m
```
From here we can see that there are different pods that are deployed.

calico-node: Calico-node runs on every Kubernetes cluster node as a DaemonSet. It is responsible for enforcing network policy, setting up routes on the nodes, plus managing any virtual interfaces for IPIP, VXLAN, or WireGuard.
calico-typha: Typha is as a stateful proxy for the Kubernetes API server. It's used by every calico-node pod to query and watch Kubernetes resources without putting excessive load on the Kubernetes API server.  The Tigera Operator automatically scales the number of Typha instances as the cluster size grows.
calico-kube-controllers: Runs a variety of Calico specific controllers that automate synchronization of resources. For example, when a Kubernetes node is deleted, it tidies up any IP addresses or other Calico resources associated with the node.
Reviewing Node Health
Finally, we can review the health of our Kubernetes nodes by invoking the kubectl command.
```
kubectl get nodes -A
```
Example output:

NAME      STATUS   ROLES    AGE   VERSION
control   Ready    master   37m   v1.18.10+k3s1
node2     Ready    <none>   16m   v1.18.10+k3s1
node1     Ready    <none>   31m   v1.18.10+k3s1
Now we can see that our Kubernetes nodes have a status of Ready and are operational. Calico is now installed on your cluster and you may proceed to the next module: Installing the Sample Application.
