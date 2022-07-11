# Deploy EKS Cluster

This lab walks through launching an EKS cluster using `eksctl`, and deploying an application using microservices. 



### PREREQUISITES

For this module, we need to download the [eksctl](https://eksctl.io/) binary:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

Confirm the `eksctl` command works:

```bash
eksctl version
```

Enable `eksctl` bash-completion

```bash
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```



### LAUNCH EKS CLUSTER

Create an eksctl deployment file (eksworkshop.yaml) use in creating your cluster using the following syntax:

```bash
cat << EOF > eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.19"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

Next, use the file you created as the input for the eksctl cluster creation.

```bash
eksctl create cluster -f eksworkshop.yaml
```

**Launching EKS and all the dependencies will take approximately 15 minutes**



### TEST THE CLUSTER

After the cluster build completes, run some tests.



#### Test the cluster:

Confirm your nodes:

```bash
kubectl get nodes # if we see our 3 nodes, we know we have authenticated correctly
```



#### Update the kubeconfig file to interact with your cluster:

```bash
aws eks update-kubeconfig --name eksworkshop-eksctl --region ${AWS_REGION}
```



#### Export the Worker Role Name for use throughout the workshop:

```bash
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

#### Congratulations!

You now have a fully working Amazon EKS Cluster that is ready to use! Before you move on to any other labs, make sure to complete the steps on the next page to update the EKS Console Credentials.



### Deploy the sample application 

Let's deploy a sample three-tier application to our new cluster. 



### DEPLOY NODEJS BACKEND API

Deploy the NodeJS Backend API.

Copy/Paste the following commands into your Cloud9 workspace:

```bash
cd ~/environment/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

We can watch the progress by looking at the deployment status:

```bash
kubectl get deployment ecsdemo-nodejs
```



You can also check the progress by running: 

```bash
kubectl get pods 
```



### DEPLOY CRYSTAL BACKEND API

Let’s bring up the Crystal Backend API

Copy/Paste the following commands into your Cloud9 workspace:

```bash
cd ~/environment/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

We can watch the progress by looking at the deployment status:

```bash
kubectl get deployment ecsdemo-crystal
```



### CHECK THE SERVICE TYPES

Before we bring up the frontend service, let’s take a look at the service types we are using: This is `kubernetes/service.yaml` for our frontend service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
spec:
  selector:
    app: ecsdemo-frontend
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```



Notice `type: LoadBalancer`: This will configure an ELB to handle incoming traffic to this service.

Compare this to `kubernetes/service.yaml` for one of our backend services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```



Notice there is no specific service type described. The default type is `ClusterIP`. This exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster.



### DEPLOY FRONTEND SERVICE

#### Challenge:

**Let’s bring up the Ruby Frontend!**

Deploy the frontend using the steps learned in this lab. If you run into problems or have questions, ask the instructor.



### FIND THE SERVICE ADDRESS

Now that we have a running service that is `type: LoadBalancer` we need to find the ELB’s address. We can do this by using the `get services` operation of kubectl:

```
kubectl get service ecsdemo-frontend
```

If the field isn’t wide enough to show the FQDN of the ELB. We can adjust the output format with this command:

```
kubectl get service ecsdemo-frontend -o wide
```

If we wanted to use the data programmatically, we can also output via JSON. This is an example of how we might be able to make use of JSON output:

```
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')

curl -m3 -v $ELB
```

**NOTE:It will take several minutes for the ELB to become healthy and start passing traffic to the frontend pods.**

You should also be able to copy/paste the Load Balancer hostname into your browser and see the application running. Keep this tab open while we scale the services up on the next page.



### SCALE THE BACKEND SERVICES

When we launched our services, we only launched one container of each. We can confirm this by viewing the running pods:

```
kubectl get deployments
```

Now let’s scale up the backend services:

```
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```

Confirm by looking at deployments again:

```
kubectl get deployments
```

Also, check the browser tab where we can see our application running. You should now see traffic flowing to multiple backend services.



### SCALE THE FRONTEND

#### Challenge:

Scale the frontend using the steps learned in this lab. If you run into problems or have questions, ask the instructor.

Check the browser tab where we can see our application running. You should now see traffic flowing to multiple frontend services.



### CLEANUP THE APPLICATIONS

To delete the resources created by the applications, we should delete the application deployments:

```bash
cd ~/environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```



### Congratulations! 

You've launched an EKS cluster and deployed a microservices application successfully.