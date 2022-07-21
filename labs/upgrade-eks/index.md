# PATCHING/UPGRADING YOUR EKS CLUSTER

As EKS tracks upstream Kubernetes that means that customers can, and should, regularly upgrade their EKS to stay within the project’s upstream support window. 

In addition to upgrades to Kubernetes, there are other related upgrades to think about with your cluster as well:

- The Amazon Machine Image (AMI) of your Nodes - including not just the portion of Kubernetes that is part of the image, the kubelet, but everything else there (OS, containerd, etc.). The control plane always supports managing kubelets that are one version behind itself (n-1) to help facilitate this upgrade.
- The foundational DaemonSets that are on deployed onto every EKS cluster (kube-proxy, CoreDNS, and the AWS CNI) may need to be upgraded as you upgrade Kubernetes.
- And any Add-ons/Controllers/Drivers that you’ve added to extend Kubernetes and provide necessary cluster functionality may need to be upgraded as you upgrade Kubernetes

In this lab, you’ll follow the AWS suggested process to upgrade your cluster, including its Managed Node Group, to get first-hand experience with this process and where EKS and Managed Node Groups help.



# THE UPGRADE PROCESS

The process goes as follows:

1. (Optional) Check if the new version you are upgrading to has any API deprecations, which will mean that you’ll need to change your YAML Spec files for them to continue to work on the new cluster. This is only the case with some version upgrades. There are various tools that can help with this such as [kube-no-trouble](https://github.com/doitintl/kube-no-trouble). Since there are no such deprecations going from 1.20 to 1.21 we’ll skip this step here.
2. Run a `kubectl get nodes` and ensure that all of your Nodes are running the current version. Kubernetes can only support nodes that are one version behind - meaning they all need to match the current Kubernetes version so when you upgrade the EKS control plane, they’ll then only be one version behind. For Fargate relaunching a Pod (maybe by deleting it and letting the ReplicaSet replace it) will bring it in line with the current Kubernetes version.
3. Upgrade the EKS Control Plane to the new major version
4. Check if the core add-ons (kubeproxy, CoreDNS, and the CNI) that ship with EKS require an upgrade to coincide with the major version upgrade. This will be in the [upgrade documentation](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html#w665aac14c15b5c17). If so, follow that documentation to upgrade those. In this case (a 1.20 to 1.21 upgrade) the documentation says we’ll need to upgrade both CoreDNS and kubeproxy.
5. Upgrade any worker nodes so that the kubelet on them (which will now be one Kubernetes major version behind) matches that of the EKS control plane. While you don’t have to do this immediately it is a good idea to have the Nodes on the same version as the control plane as soon as is practical - plus, it will make Step 2 easier the next time you have to upgrade. If you are using Managed Node Groups (as we are here), then EKS can help facilitate this with a safe automated process that orchestrates both the AWS and Kubernetes side of a rolling Node replacement/upgrade. If you are using Fargate, then this will happen automatically the next time your Pods are replaced.



# UPGRADE EKS CONTROL PLANE

The first step of this process is to upgrade the EKS Control Plane.

Since we used `eksctl` to provision our cluster we’ll use that tool to do our upgrade as well.

First we’ll run this command

```bash
eksctl upgrade cluster --name=eksworkshop-eksctl
```

You’ll see in the output that it found our cluster, worked out that it is 1.19 and the next version is 1.20 (you can only upgrade one version at a time with EKS) and that everything is ready for us to proceed with an upgrade.

```
$ eksctl upgrade cluster --name=eksworkshop-eksctl
2022-07-21 17:48:52 [ℹ]  (plan) would upgrade cluster "eksworkshop-eksctl-demo" control plane from current version "1.19" to "1.20"
2022-07-21 17:48:52 [ℹ]  re-building cluster stack "eksctl-eksworkshop-eksctl-demo-cluster"
2022-07-21 17:48:52 [✔]  all resources in cluster stack "eksctl-eksworkshop-eksctl-demo-cluster" are up-to-date
2022-07-21 17:48:52 [ℹ]  checking security group configuration for all nodegroups
2022-07-21 17:48:52 [ℹ]  all nodegroups have up-to-date cloudformation templates
2022-07-21 17:48:52 [!]  no changes were applied, run again with '--approve' to apply the changes
```

We’ll run it again with an `--approve` appended to proceed

```bash
eksctl upgrade cluster --name=eksworkshop-eksctl --approve
```

This process should take approximately 25 minutes. You can continue to use the cluster during the control plane upgrade process, but you might experience minor service interruptions. For example, if you attempt to connect to one of the EKS API servers just before or just after it’s terminated and replaced by a new API server running the new version of Kubernetes, you might experience temporary API call errors or connectivity issues. If this happens, retry your API operations until they succeed. Your existing Pods/workloads running in the data plane should not experience any interruption during the control plane upgrade.

## Upgrade the cluster to the latest version

Now run the upgrade command again to upgrade from 1.20 to 1.21



# UPGRADE EKS CORE ADD-ONS

When you provision an EKS cluster you get three add-ons that run on top of the cluster and that are required for it to function properly:

- kubeproxy
- CoreDNS
- aws-node (AWS CNI or Network Plugin)

Looking at the the [upgrade documentation](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html#w665aac14c15b5c17) for our upgrade, we see that we’ll need to upgrade the kubeproxy and CoreDNS. In addition to performing these steps manually with kubectl as documented there you’ll find that `eksctl` can do it for you as well.

Since we are using `eksctl` in the lab we’ll run the two necessary commands for it to do these updates for us:

```bash
eksctl utils update-kube-proxy --cluster=eksworkshop-eksctl --approve
```

and then

```bash
eksctl utils update-coredns --cluster=eksworkshop-eksctl --approve
```

We can confirm we succeeded by retrieving the versions of each with the commands:

```bash
kubectl get daemonset kube-proxy --namespace kube-system -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
kubectl describe deployment coredns --namespace kube-system | grep Image | cut -d "/" -f 3
```



# UPGRADE MANAGED NODE GROUP

Finally, we have gotten to the last step of the upgrade process, which is upgrading our Nodes.

There are two ways to provision and manage your worker nodes - self-managed node groups and managed node groups. In this lab, `eksctl` was configured to use the managed node groups. This was helpful here as managed node groups make this easier for us by automating both the AWS and the Kubernetes side of the process.

The way that managed node groups do this is:

1. Amazon EKS creates a new Amazon EC2 launch template version for the Auto Scaling group associated with your node group. The new template uses the target AMI for the update.
2. The Auto Scaling group is updated to use the latest launch template with the new AMI.
3. The Auto Scaling group maximum size and desired size are incremented by one up to twice the number of Availability Zones in the Region where the Auto Scaling group is deployed. This is to ensure that at least one new instance comes up in every Availability Zone in the Region where your node group is deployed.
4. Amazon EKS checks each node in the node group for the `eks.amazonaws.com/nodegroup-image` label and applies an `eks.amazonaws.com/nodegroup=unschedulable:NoSchedule` taint on all of the nodes in the node group that aren’t labeled with the latest AMI ID. This prevents nodes that have already been updated from a previous failed update from being tainted.
5. Amazon EKS randomly selects a node in the node group and evicts all pods from it.
6. After all of the pods are evicted, Amazon EKS cordons the node. This is done so that the service controller doesn’t send any new requests to this node and removes this node from its list of healthy, active nodes.
7. Amazon EKS sends a termination request to the Auto Scaling group for the cordoned node.
8. Steps 5-7 are repeated until there are no nodes in the node group that are deployed with the earlier version of the launch template.
9. The Auto Scaling group maximum size and desired size are decremented by 1 to return to your pre-update values.

**NOTE: If we instead had used a self-managed node group, then we need to do the Kubernetes taint and draining steps ourselves to ensure Kubernetes knows that Node is going away and can manage that process gracefully in order for such an upgrade to be non-disruptive.**



We trigger the Managed Node Group upgrade process by running the following `eksctl` command:

```bash
eksctl upgrade nodegroup --name=nodegroup --cluster=eksworkshop-eksctl --kubernetes-version=1.20
```

In another Terminal tab, you can follow the progress with:

```bash
kubectl get nodes --watch
```

You’ll notice the new nodes come up (one in each AZ), each node  **STATUS** `SchedulingDisabled`, then eventually that node is deleted and a new node is deployed to replace it and so on as described in the process above until all the nodes have been upgraded. After the upgrade is complete it’ll scale back down from 6 nodes to the original 3.

## Upgrade the cluster to the latest version

Now go through the same process and upgrade the control plane and managed node group to version 1.21.



## Congrats!

You've completed the lab!