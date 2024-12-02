## AKS Cluster Upgrade Process Flow


This document will cover the upgrade process flow of a Kubernetes Cluster. This document will also talk about the scenarios, strategies like when and how the upgrade will happen and the buisness approvals for approving the cluster upgrade.

* [AKS Environment](https://github.com/procter-gamble/k8sCICD/blob/feat_node_pool_desc/cluster_upgrade_process.md#aks-environment)
* [Development VM](https://github.com/procter-gamble/k8sCICD/blob/feat_node_pool_desc/cluster_upgrade_process.md#development-vm)
* [Upgrade Scenario](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#upgrade-scenario)
* [Approvals](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#approvals)
* [Upgrade Strategy](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#upgrade-strategy)
   * [Reduce disruption to existing workloads during an upgrade](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#reduce-disruption-to-existing-workloads-during-an-upgrade)
   * [Set your tolerance for disruption](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#set-your-tolerance-for-disruption)
   * [Sufficient IP addresses](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#sufficient-ip-addresses)
   * [Node Pool](https://github.com/procter-gamble/k8sCICD/blob/feat_node_pool_desc/cluster_upgrade_process.md#node-pool)
   * [New Node Pool](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#new-node-pool)
     * [Set environment vars](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#set-environment-vars)
     * [Adding a new node pool](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#adding-a-new-node-pool)
     * [Updgrade Control Plane](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#updgrade-control-plane)
     * [Upgrade Node Pool](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#upgrade-node-pool)
     * [Uncordon All Nodes](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#uncordon-all-nodes)
     * [Delete the Node Pool](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#delete-the-node-pool)
* [Backup Plan](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#backup-plan)
* [Post Upgrade Notification](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#post-upgrade-notification)
* [References](https://github.com/procter-gamble/k8sCICD/blob/feat_upgrade_process_flow/cluster_upgrade_process.md#references)

### AKS Environment

ClusterName: [aks-dev-cluster](https://portal.azure.com/#@pgone.onmicrosoft.com/resource/subscriptions/ed6d71e6-0262-49ad-b8ff-074000058889/resourceGroups/AZ-RG-KubernetesCICD-Dev-01/providers/Microsoft.ContainerService/managedClusters/aks-dev-cluster/overview)

### Development VM


To access the cluster a virtual machine is being used where all the dependencies for kubernetes have been installed like ```kubectl``` ans so on. This VM will be used for other development activities.

[azl-dev-vm](https://portal.azure.com/#@pgone.onmicrosoft.com/resource/subscriptions/ed6d71e6-0262-49ad-b8ff-074000058889/resourceGroups/AZ-RG-TerraformModules-Dev-02/providers/Microsoft.Compute/virtualMachines/azl-dev-vm/overview)

To access the cluster inside the vm use the following command.

```sh
kubectl config use-context aks-dev-cluster
```

### Upgrade Scenario:

Kubernetes often releases updates, to deliver security updates, fix known issues, and introduce new features.  It is required to upgrade the cluster to keep up with the latest security features and bug fixes, as well as benefit from new features being released on an on-going basis.

<i>**Note**: When you are upgrading a Kubernetes cluster, the flow should be from version 1.22.x to version 1.23.x, and from version 1.23.x to 1.23.y (where y > x). Skipping `MINOR` versions when upgrading is unsupported.</i>
  
### Approvals:
  
Notifying the cluster owners/change approvers about the approvals before the upgrade is important. The approvals can be sent via email to the respective owners/approvers.

### Upgrade Strategy:

Before getting into the cluster upgrade, with any upgrade/update there is always likelihood things might fail so you need to have multiple strategies, here are few suggestions.

#### Reduce disruption to existing workloads during an upgrade:

To increase upgrade predictability and to align upgrades with off-peak business hours, you can control the upgrades of both the control plane and nodes by creating a maintenance window. 

#### Set your tolerance for disruption

Ensure that any PodDisruptionBudgets (PDBs) allow for at least 1 pod replica to be moved at a time otherwise the drain/evict operation will fail. If the drain operation fails, the upgrade operation will fail by design to ensure that the applications are not disrupted.

#### Sufficient IP addresses

Please make sure there are sufficient private IP addresses within the AKS associated subnet if the cluster is running with Azure CNI. This is critical when you add a new node pool during the upgrade and IP addresses required for each nodes within the pool.

#### Node Pool

In Azure Kubernetes Service (AKS), nodes of the same configuration are grouped together into node pools. These node pools contain the underlying VMs that run your applications. Following are the list of nodes available in dev cluster to manage self hosted runners.

| Name   |     Description    |
|------|---------|
| **default** |    Default nodepool |
| **adoworkload**, **ghworkload** | nodepool that runs containerd as runtime to handle self hosted runners for azure devops | 
| **ghrunner**, **ghrunnercrio** | nodepool than runs crio as runtime to handle self hosted for Github Actions |



#### New Node Pool

To minimise risk creating a new node pool in the same cluster and then draining older one. Here we enhance the cluster with an addional nodepool, which will host the running services, while we are upgrading the first nodepool onto a higher version.

##### Set environment vars

```sh
$ export RESOURCE_GROUP=YOUR_GROUP_NAME; export CLUSTER_NAME=CLUSTER_NAME
```

##### Adding a new node pool

```sh
$ az aks nodepool add --resource-group $RESOURCE_GROUP --cluster-name $CLUSTER_NAME --node-vm-size Standard_DS2_v2 --name update-pool --node-count 2
```

Now the cluster is running with 2 node pools, the next step is to shift all running services onto the new cluster nodes that was created.

```sh
$ kubectl get nodes -o wide

NAME                                  STATUS   ROLES   AGE   VERSION
aks-update-pool-xxxxxxxx-vmss000000   Ready    agent   9d    v1.20.15
aks-update-pool-xxxxxxxx-vmss000001   Ready    agent   9d    v1.20.15
aks-default-xxxxxxxx-vmss000001       Ready    agent   9d    v1.20.15
aks-default-xxxxxxxx-vmss000002       Ready    agent   9d    v1.20.15
aks-default-xxxxxxxx-vmss000004       Ready    agent   9d    v1.20.15
```

Now let's disable the nodes on the first node pool, all pods get evicted and are restarted on the other node pool. Using below commands drain all the nodes from the default node pool.

```sh
$ kubectl drain aks-default-xxxxxxxx-vmss000001
```

Now all pods will get evicted from the first nodepool and get restarted on the other nodepool, enabling a safe upgrade of the unused nodes in the first pool.

##### Updgrade Control Plane

The next step is to upgrade the control plane onto the new version. To find the supported upgrades that are available, run the below command.

```sh
$ az aks get-upgrades --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

Pick your version:
```sh
$ az aks upgrade --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --kubernetes-version 1.22.15 --control-plane-only
```

##### Upgrade Node Pool

After having upgraded the control plane, we can upgrade the first node pool.

```sh
$ az aks nodepool upgrade --resource-group $RESOURCE_GROUP --cluster-name $CLUSTER_NAME --kubernetes-version 1.22.15 --name default
```

##### Uncordon All Nodes

The next step is just uncordon all nodes in the node pool default and drain each node in node pool internal.

<i>**Note**: The upgrade process will anyways uncordons all nodes in the upgraded pool, just check that and then drain each node in the other pool. This way all your pods will migrate back to the default pool, which is now running on the higher version.</i>

##### Delete the Node Pool

Finally you can delete or upgrade the node pool ```update-pool```.

<i>**Note**: Please be aware that you need at least one node pool in the status of ```System``` , otherwise you won’t be able to delete the other one.</i>

### Backup Plan

Incase of upgrade failed at some point, analyse the reasons and fix the error and restart the upgrade process all over again. Make sure to take a backup of all the application deployment manifests incase if you want to redeploy them all over again. There are other approaches that you can consider where we can backup and restore the entire cluster using tools like ```Velero```.

### Post Upgrade Notification

Post the successful upgrade, an email notification should be sent accross to the cluster owners/approvers about the status of the application with the new K8 API version.

### References

* Upgrade AKS Cluster - https://learn.microsoft.com/en-us/azure/aks/upgrade-cluster?tabs=azure-cli
* Upgrading and Patching AKS - https://www.kunalbabre.com/upgrading-and-patching-aks/
* Velero - https://github.com/mutazn/Backup-and-Restore-AKS-cluster-using-Velero
* Pod Disruption Budget - https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-scheduler#plan-for-availability-using-pod-disruption-budgets







