# Cluster API Demo Notes

There are many prereqs for this demo. I try to call them all out, but please reference the CAPI docs for the most up to date info

https://cluster-api.sigs.k8s.io/user/quick-start.html

## Setup local kind cluster to use as Management Cluster

```
$ cd ~/demo/capi-demo/
$ kind create cluster --name=cluster-api-demo
# Not needed with newer kind versions
#$ export KUBECONFIG="$(kind get kubeconfig-path --name="cluster-api-demo")"
$ kubectl cluster-info; kubectl get nodes -o wide
```

## Init the Management Cluster

```
$ clusterctl init
$ kubectl get pods -A | grep capi
```

## Install AWS Workload Cluster

### Source ENV Vars for AWS

```
$ source ./aws/.env
```

NOTE: More info on what's required is located here: https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/master/docs/prerequisites.md

### Setup AWS IAM stuff

You will need to download the `clusterawsadm` binary release from https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases

```
$ clusterawsadm alpha bootstrap create-stack
$ export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm alpha bootstrap encode-aws-credentials)

```

### Init AWS Infrastructure Provider

```
$ clusterctl init --infrastructure aws
$ kubectl get pods -n capa-system
```

## Provision Workload Cluster

### Setup Namespace

```
$ kubectl create ns aws-clusters
```

### Deploy `cluster` resource - AWS

```
$ clusterctl config cluster aws-cluster2 --target-namespace aws-clusters --infrastructure aws --kubernetes-version v1.17.3 --control-plane-machine-count=1 --worker-machine-count=1 > ./aws/cluster2/aws-cluster2.yaml
$ kubectl apply -f ./aws/cluster2/aws-cluster2.yaml -n aws-clusters
$ kubectl get clusters -n aws-clusters
$ kubectl get kubeadmcontrolplanes -n aws-clusters
$ kubectl get machinedeployments -n aws-clusters
$ kubectl get machines -n aws-clusters
```

NOTE: This will take several minutes
    - Setting up all the networking (VPC, security groups, ELB, etc.)
    - Provisioning storage
    - Initializing each instance

### Get Kubeconfig - AWS

```
$ kubectl -n aws-clusters get secret/aws-cluster2-kubeconfig -o jsonpath={.data.value} \
  | base64 --decode \
  > ./aws/cluster2/aws-cluster2.kubeconfig
```
NOTE: direnv should be setup to autoswitch to this kubeconfig when you navigate to the `./aws/cluster2` directory

Check out node status

```
$ cd ./aws/cluster2
$ kubectl get nodes -o wide
```

NOTE: Node status will show as `NotReady` until a CNI plugin is installed

### Deploy CNI - AWS

```
$ kubectl apply -f https://docs.projectcalico.org/v3.12/manifests/calico.yaml
$ kubectl get pods -A
$ kubectl get nodes -o wide
# View status from the Management Cluster
$ cd ../../
$ kubectl get clusters -n aws-clusters
$ kubectl get kubeadmcontrolplanes -n aws-clusters
$ kubectl get machinedeployments -n aws-clusters
$ kubectl get machines -n aws-clusters
```

NOTE: Node status should now show `Ready`
NOTE: Once the cluster nodes are marked `Ready`, their associated machine status on the management cluster should now reflect as `Running`

## Deploy Sample Workload - AWS

Deploy httpbin app to show normal k8s operation

```
cd ./aws/cluster2
$ kubectl apply -f ../../sample-apps/httpbin-deploy.yaml
$ kubectl get pods
$ kubectl get pods -A
```

## Show Worker Node Scaling - AWS

Scale existing Machine Deployment

```
$ cd ../../
$ kubectl get md -n aws-clusters aws-cluster2-md-0
$ kubectl scale md -n aws-clusters aws-cluster2-md-0 --replicas=3
$ kubectl get md -n aws-clusters aws-cluster2-md-0
# View things from the Workload Cluster
$ cd ./aws/cluster2
$ kubectl get nodes -o wide
# View things again from the Management Cluster
$ cd ../../
$ kubectl get md -n aws-clusters aws-cluster2-md-0
```

## Upgrade Worker Nodes - AWS

Edit K8s Version field within the Machine Deployment....and wait....

```
$ kubectl edit md -n aws-clusters aws-cluster2-md-0
$ kubectl get md -n aws-clusters aws-cluster2-md-0
$ k get machines -n aws-clusters -o custom-columns=NAME:.metadata.name,PROVIDERID:.spec.providerID,PHASE:.status.phase,K8S\ VERSION:.spec.version
$ cd ./aws/cluster2
$ kubectl get nodes -o wide
```

NOTE: Change version from 1.17.3 to 1.17.5

---

## Install Azure Workload Cluster

### Source ENV Vars for Azure

```
$ source ./azure/.env
```

NOTE: More info on what's required is located here: https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/master/docs/getting-started.md#prerequisites

### Init Azure Infrastructure Provider

```
$ clusterctl init --infrastructure azure
$ kubectl get pods -n capz-system
```

### Setup Namespace

```
$ kubectl create ns azure-clusters
```

### Deploy `cluster` resource - Azure

```
$ clusterctl config cluster azure-cluster1 --target-namespace azure-clusters --infrastructure azure --kubernetes-version v1.17.3 --control-plane-machine-count=3 --worker-machine-count=3 > ./azure/cluster1/azure-cluster1.yaml
$ kubectl apply -f ./azure/cluster1/azure-cluster1.yaml -n azure-clusters
$ kubectl get clusters -n azure-clusters
$ kubectl get kubeadmcontrolplanes -n azure-clusters
$ kubectl get machinedeployments -n azure-clusters
$ kubectl get machines -n azure-clusters
```

NOTE: The cluster status/phase will show as `provisioning` until you deploy a node
NOTE: The EC2 Portal will show 1 instances created, the bastion node. The bastion node is used to bridge public -> private networks so your cluster can be 100% private networking

### Get Kubeconfig - Azure

```
kubectl --namespace=azure-clusters get secret azure-cluster1-kubeconfig -o json \
  | jq -r .data.value \
  | base64 --decode \
  > ./azure/cluster1/azure-cluster1.kubeconfig
```
NOTE: direnv should be setup to autoswitch to this kubeconfig when you navigate to the `./aazure` directory

Check out node status

```
$ cd ./azure/cluster1
$ kubectl get nodes
```

NOTE: Node status will show as `NotReady` until a CNI plugin is installed

### Deploy CNI - Azure

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-azure/master/templates/addons/calico.yaml
$ kubectl get pods -A
$ kubectl get nodes -o wide
# View status from the Management Cluster
$ cd ../../
$ kubectl get clusters -n azure-clusters
$ kubectl get kubeadmcontrolplanes -n azure-clusters
$ kubectl get machinedeployments -n azure-clusters
$ kubectl get machines -n azure-clusters
```

NOTE: Azure does not currently support Calico networking. As a workaround, it is recommended that Azure clusters use the Calico spec below that uses VXLAN.

## Deploy Sample Workload - Azure

Deploy httpbin app to show normal k8s operation

```
$ cd ./azure/cluster1
$ kubectl apply -f ../../sample-apps/httpbin-deploy.yaml
$ kubectl get pods
$ kubectl get pods -A
```

---

#############################


## Things to show:

All commands to be run on management cluster

### Show general CAPI primitives
```
$ kubectl get pods -A
$ kubectl get clusters -A
$ kubectl get machines -A
$ kubectl get md -A
$ kubectl get kcp -A
```

### Show scaling and watch controller logs

```
$ kubectl scale md -n aws-clusters aws-cluster1-md-0 --replicas 3
$ kubectl logs -n capa-system -l control-plane=capa-controller-manager -c manager -f
```

### Show Worker Node Upgrade

```
$ kubectl edit md -n azure-clusters azure-cluster1-md-0
```
