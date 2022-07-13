# USING FARGATE AND APP MESH

## DEPLOY PRODUCT CATALOG APP

To understand `AWS App Mesh`, it's best to also understand any applications that run on top of it. In this section, we’ll walk you through the following parts of application setup and deployment:

- Discuss the Application Architecture
- Build the Application Services container images
- Push the container images to Amazon ECR
- Deploy the application services into the Amazon EKS cluster, initially **without** AWS App Mesh.
- Test the connectivity between the Services



### ABOUT PRODUCT CATALOG APPLICATION

#### Application Architecture

The example application architecture we’ll walk you through creating on App Mesh is called `Product Catalog` and is used in any eCommerce Application. We have attempted to pick different technology frameworks for building applications to include the polyglot nature of microservices-based applications.

This application is composed of three microservices:

[![Product Catalog App without App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/no-appmesh-arch.png)](https://www.eksworkshop.com/images/app_mesh_fargate/no-appmesh-arch.png)

- Frontend

  - Frontend service `frontend-node` shows the UI for the Product Catalog functionality
  - Developed in Nodejs with ejs templating
  - Deployed to `EKS Managed Nodegroup`

- Product Catalog Backend

  - Backend service `prodcatalog` is a Rest API Service that performs following operations:

    - Add a Product into Product Catalog

    - Get the Product from the Product Catalog

    - Calls Catalog Detail backend service `proddetail` to get Product Catalog Detail information like vendors

    - Get all the products from the Product Catalog

      ```
      /products
      
      {
          "products": {
              "1": "Table",
              "2": "Chair"
          },
          "details": {
              "version": "2",
              "vendors": [
                  "ABC.com",
                  "XYZ.com"
              ]
          }
      }
      ```

  - Developed in Python Flask Restplus with Swagger UI for Rest API

  - Deployed to `EKS Fargate`

- Catalog Detail Backend

  - Backend service `proddetail`is a Rest API Service that performs the following operation:

    - Get catalog detail which includes version number and vendor names

      ```
      /catalogDetail
      
      {
          "version": "2",
          "vendors": [
              "ABC.com",
              "XYZ.com"
          ]
      }
      ```

  - Developed in Nodejs

  - Deployed to `EKS Managed Nodegroup`

From the above diagram, the service-call relationship between the services in the `Product Catalog` application can be summarized as:

- Frontend `frontend-node` »»> calls »»> Product Catalog backend `prodcatalog`.
- Product Catalog backend `prodcatalog`  > calls > Catalog Detail backend `proddetail`.



### CREATE THE PRODUCT CATALOG APPLICATION

Let’s create the Product Catalog Application!

#### Build Application Services

Build and Push the Container images to ECR for all the three services

```bash
cd ~/environment/eks-app-mesh-polyglot-demo
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
PROJECT_NAME=eks-app-mesh-demo
export APP_VERSION=1.0
for app in catalog_detail product_catalog frontend_node; do
  aws ecr describe-repositories --repository-name $PROJECT_NAME/$app >/dev/null 2>&1 || \
  aws ecr create-repository --repository-name $PROJECT_NAME/$app >/dev/null
  TARGET=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$PROJECT_NAME/$app:$APP_VERSION
  docker build -t $TARGET apps/$app
  docker push $TARGET
done
```

Building/Pushing Container images first time to ECR may take around 3-5 minutes

Once completed, you can confirm the images are in ECR by logging into the console[![ecr](https://www.eksworkshop.com/images/app_mesh_fargate/ecr.png)](https://www.eksworkshop.com/images/app_mesh_fargate/ecr.png)

#### Deploy the Application Services to EKS

```bash
envsubst < ./deployment/base_app.yaml | kubectl apply -f -


deployment.apps/prodcatalog created
service/prodcatalog created
deployment.apps/proddetail created
service/proddetail created
deployment.apps/frontend-node created
service/frontend-node created
```

Fargate pod creation for `prodcatalog` service may take 3 to 4 minutes

#### Get the deployment details

```bash
kubectl get deployment,pods,svc -n prodcatalog-ns -o wide
```

You can see that:

- Product Catalog service was deployed to `Fargate` pod as it matched the configuration (namespace `prodcatalog-ns` and pod spec label as `app=prodcatalog`) that we had specified when creating fargate profile

- And other services Frontend and Catalog Product Detail were deployed into `Managed Nodegroup`

  ```
  NAME                            READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS      IMAGES                                                                          SELECTOR
  deployment.apps/frontend-node   1/1     1            1           44h   frontend-node   $ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/frontend-node:4.6                  app=frontend-node
  deployment.apps/prodcatalog     1/1     1            1           22h   prodcatalog     $ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/product-catalog:1.2                app=prodcatalog
  deployment.apps/proddetail      1/1     1            1           44h   proddetail      $ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/product-detail:1.1                 app=proddetail
  
  NAME                                 READY   STATUS    RESTARTS   AGE   IP               NODE                                                   NOMINATED NODE   READINESS GATES
  pod/frontend-node-77d64585d4-xxxx   1/1     Running   0          13h   192.168.X.6     ip-192-168-X-X.us-west-2.compute.internal           <none>           <none>
  pod/prodcatalog-98f7c5f87-xxxxx      1/1     Running   0          13h   192.168.X.17   fargate-ip-192-168-X-X.us-west-2.compute.internal   <none>           <none>
  pod/proddetail-5b558df99d-xxxxx      1/1     Running   0          18h   192.168.24.X   ip-192-168-X-X.us-west-2.compute.internal            <none>           <none>
  
  NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)        AGE   SELECTOR
  service/frontend-node   ClusterIP      10.100.X.X    <none>                                                                          9000/TCP       44h   app=frontend-node
  service/prodcatalog     ClusterIP      10.100.X.X   <none>                                                                          5000/TCP       41h   app=prodcatalog
  service/proddetail      ClusterIP      10.100.X.X   <none>                                                                          3000/TCP       44h   app=proddetail                                                               3000/TCP       103m
      
  ```

#### Confirm that the fargate pod is using the Service Account role

```bash
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl describe pod ${BE_POD_NAME} -n  prodcatalog-ns | grep 'AWS_ROLE_ARN\|AWS_WEB_IDENTITY_TOKEN_FILE\|serviceaccount' 
```

You should see the below output which has the same role that we had associated with the Service Account as part of Fargate setup.

```
AWS_ROLE_ARN:                 arn:aws:iam::$ACCOUNT_ID:role/eksctl-eksworkshop-eksctl-addon-iamserviceac-Role1-1PWNQ4AJFMVBF
AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
/var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
/var/run/secrets/kubernetes.io/serviceaccount from prodcatalog-envoy-proxies-token-69pql (ro)
```



#### Confirm that the fargate pod logging is enabled

```bash
kubectl describe pod ${BE_POD_NAME} -n  prodcatalog-ns | grep LoggingEnabled
```

We can see the confirmation in the events that says Successfully enabled logging for pod.

```
  Normal  LoggingEnabled  2m7s  fargate-scheduler  Successfully enabled logging for pod
```



#### TEST THE APPLICATION

#### Testing the Connectivity between Fargate and Nodegroup pods

To test if our ported Product Catalog App is working as expected, we’ll first exec into the `frontend-node` container.

```bash
export FE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=frontend-node -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${FE_POD_NAME} -c frontend-node bash
```

You will see a prompt from within the `frontend-node` container.

```
root@frontend-node-9d46cb55-XXX:/usr/src/app#
```

curl to Fargate `prodcatalog` backend endpoint and you should see the below response

```bash
curl http://prodcatalog.prodcatalog-ns.svc.cluster.local:5000/products/ 
```



A successful connection will output this: 

```json
{
    "products": {},
    "details": {
        "version": "1",
        "vendors": [
            "ABC.com"
        ]
    }
}
```



Exit from the `frontend-node` container exec bash. Now, To test the connectivity from Fargate service `prodcatalog` to Nodegroup service `proddetail`, we’ll first exec into the `prodcatalog` container.

```bash
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${BE_POD_NAME} -c prodcatalog bash
```

You will see a prompt from within the `prodcatalog` container.



```
root@prodcatalog-98f7c5f87-xxxxx:/usr/src/app#
```

curl to Nodegroup `proddetail` backend endpoint. 

```bash
curl http://proddetail.prodcatalog-ns.svc.cluster.local:3000/catalogDetail
```



You should see the below response. You can now exit from `prodcatalog` exec bash.

```
{"version":"1","vendors":["ABC.com"]}
```



Congratulations on deploying the initial Product Catalog Application architecture!

Before we create the App Mesh-enabled versions of the Product Catalog App, we’ll first install the `AWS App Mesh Controller for Kubernetes` into our cluster.



# APP MESH INSTALLATION

This tutorial guides you through the installation and use of the open source [**AWS App Mesh Controller for Kubernetes**](https://github.com/aws/aws-app-mesh-controller-for-k8s). AWS App Mesh Controller For K8s is a controller to help manage [AWS App Mesh](https://docs.aws.amazon.com/app-mesh/latest/userguide/what-is-app-mesh.html) resources for a Kubernetes cluster and injecting sidecars into Kubernetes Pods.

The controller maintains the custom resources (CRDs): Mesh, VirtualNode, VirtualService, VirtualRouter, VirtualGateway and GatewayRoute. When using the App Mesh Controller, you manage these App Mesh custom resources such as `VirtualService` and `VirtualNode` through the Kubernetes API the same way you manage native Kubernetes resources such as `Service` and `Deployment`.

The controller monitors your Kubernetes objects, and when App Mesh resources are created or changed it reflects those changes in AWS App Mesh for you. Specification of your App Mesh resources is done through the use of Custom Resource Definitions (CRDs) provided by the App Mesh Controller project. These custom resources map to below App Mesh API objects.

- [Mesh](https://docs.aws.amazon.com/app-mesh/latest/userguide/meshes.html)
- [VirtualService](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual_services.html)
- [VirtualNode](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual_nodes.html)
- [VirtualRouter](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual_routers.html)
- [VirtualGateway](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual_gateways.html)
- [GatewayRoute](https://docs.aws.amazon.com/app-mesh/latest/userguide/gateway-routes.html)

For a pod in your application to join a mesh, it must have an open source [Envoy](https://www.envoyproxy.io/) proxy container running as a sidecar container within the pod. This establishes the data plane that AWS App Mesh controls. So we must run an Envoy container within each pod of the Product Catalog App deployment. App Mesh uses the Envoy sidecar container as a proxy for all ingress and egress traffic to the primary microservice. Using this sidecar pattern with Envoy we create the backbone of the service mesh, without impacting our applications.

The controller will handle routine App Mesh tasks such as creating and injecting Envoy proxy containers into your application pods. Automated sidecar injection is controlled by enabling a webhook on a per-namespace basis.[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/pcapp.png)](https://www.eksworkshop.com/images/app_mesh_fargate/pcapp.png)

Our Product Catalog Application services use the below App Mesh Configuration and we will explore how to create these in detail.[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/appmeshconfig.png)](https://www.eksworkshop.com/images/app_mesh_fargate/appmeshconfig.png)



# INSTALL AWS APP MESH CONTROLLER

#### Install App Mesh Helm Chart

Add the EKS Helm Repo and install The AWS App Mesh Controller for Kubernetes using Helm. 

```bash
helm repo add eks https://aws.github.io/eks-charts

"eks" has been added to your repositories
```

#### Install App Mesh Controller

Create the namespace `appmesh-system`, enable OIDC and create IRSA (IAM for Service Account) for AWS App Mesh installation

```bash
# Create the namespace
kubectl create ns appmesh-system

# Install the App Mesh CRDs
kubectl apply -k "github.com/aws/eks-charts/stable/appmesh-controller//crds?ref=master"

# Download the IAM policy for AWS App Mesh Kubernetes Controller
curl -o controller-iam-policy.json https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/config/iam/controller-iam-policy.json

# Create an IAM policy called AWSAppMeshK8sControllerIAMPolicy
aws iam create-policy \
    --policy-name AWSAppMeshK8sControllerIAMPolicy \
    --policy-document file://controller-iam-policy.json

# Create an IAM role for the appmesh-controller service account
eksctl create iamserviceaccount --cluster eksworkshop-eksctl \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSAppMeshK8sControllerIAMPolicy  \
    --override-existing-serviceaccounts \
    --approve
```

Update the Helm Repo

```bash
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "eks" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Install App Mesh Controller into the `appmesh-system` namespace

```bash
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller \
    --set tracing.enabled=true \
    --set tracing.provider=x-ray
    
Release "appmesh-controller" has been upgraded. Happy Helming!     
NAME: appmesh-controller     
LAST DEPLOYED: Wed Jan 20 21:07:01 2021     
NAMESPACE: appmesh-system     
STATUS: deployed     
REVISION: 1    
TEST SUITE: None     
NOTES:     
AWS App Mesh controller installed! 
```

Confirm that the controller version is v1.0.0 or later.

```bash
kubectl get deployment appmesh-controller \
    -n appmesh-system \
    -o json  | jq -r ".spec.template.spec.containers[].image" | cut -f2 -d ':'
    
v1.3.0
```

Confirm all the App Mesh CRDs are created in the Cluster

```bash
kubectl get crds | grep appmesh

gatewayroutes.appmesh.k8s.aws                2020-11-02T16:02:14Z
meshes.appmesh.k8s.aws                       2020-11-02T16:02:15Z
virtualgateways.appmesh.k8s.aws              2020-11-02T16:02:15Z
virtualnodes.appmesh.k8s.aws                 2020-11-02T16:02:15Z
virtualrouters.appmesh.k8s.aws               2020-11-02T16:02:15Z
virtualservices.appmesh.k8s.aws              2020-11-02T16:02:15Z    
```

Get all the resources created in the `appmesh-system` Namespace

```bash
kubectl -n appmesh-system get all 

NAME                                     READY   STATUS    RESTARTS   AGE     
pod/appmesh-controller-fcc7c4ffc-mldhk   1/1     Running   0          47s 
NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     
service/appmesh-controller-webhook-service   ClusterIP   10.100.xx.yy   <none>        443/TCP   27m å   
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE     
deployment.apps/appmesh-controller   1/1     1            1           27m 
NAME                                            D kubectl get crds ESIRED   CURRENT   READY   AGE    
replicaset.apps/appmesh-controller-fcc7c4ffc    1         1         1       47s 
```

Congratulations on installing the AWS App Mesh Controller in your EKS Cluster!



# PORTING THE PRODUCT CATALOG TO APP MESH

#### Challenge

Today, the Product Catalog frontend `frontend-node` is hardwired to make requests to `prodcatalog` and `prodcatalog` is hardwired to make requests to `proddetail`.

Each time there is a new version of `proddetail` released, we also need to release a new version of `prodcatalog` to support both the new and the old version to point to its version-specific endpoints. It works, but it’s not an optimal configuration to maintain for the long term.

The `prodcatalog` backend service is deployed to Fargate, and the rest of the services, `frontend-node` and `proddetail` are deployed to the Managed Nodegroup, we need to add all these services into the App Mesh and ensure these microservices can communicate with each other.

#### Solution

We’re going to demonstrate how `AWS App Mesh` can be used to simplify this architecture; by virtualizing the `proddetail` service, we can add dynamic configuration and route traffic to the versioned endpoints of our choosing, minimizing the need for complete re-deployment of the `prodcatalog` service each time there is a new `proddetail` service release.

We’re also going to demonstrate how all the microservices in Nodegroup and Fargate can communicate with each other via App Mesh.

When we’re done with porting the application in this chapter, our app will look more like the following.

[![Product Catalog App with App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/appmeshv1-1.png)](https://www.eksworkshop.com/images/app_mesh_fargate/appmeshv1-1.png)

Now that the App Mesh Controller and CRDs are installed, we’re ready to define the App Mesh components required for our mesh-enabled version of the app.

# MESH RESOURCES AND DESIGN

#### App Mesh Design

[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/meshify.png)](https://www.eksworkshop.com/images/app_mesh_fargate/meshify.png)

In the image above you see all the services in the Product Catalog Application are running within App Mesh. Each of the services has a `VirtualNode` defined (`frontend-node`, `prodcatalog`, and `proddetail-v1`), as well as `VirtualService` (`frontend-node`, `prodcatalog` and `proddetail`). These `VirtualServices` send traffic to `VirtualRouters` within the mesh, which in turn specify routing rules. This drives traffic to their respective `VirtualNodes` and ultimately to the service endpoints within Kubernetes.

##### How will it be different using App Mesh?

- Functionally, the mesh-enabled version will do exactly what the current version does;
  - requests made by `frontend-node` will be served by the `prodcatalog` backend service;
  - requests made by `prodcatalog` will be served by the `proddetail-v1` backend service;
- The difference will be that we’ll use `AWS App Mesh` to create new Virtual Services called `prodcatalog` and `proddetail`
  - Requests made by `frontend-node` service will logically send traffic to `VirtualRouter` instances which will be configured to route traffic to the service endpoints within your cluster, to `prodcatalog`.
  - Requests made by `prodcatalog` service will logically send traffic to `VirtualRouter` instances which will be configured to route traffic to the service endpoints within your cluster, to `proddetail-v1`.

#### App Mesh Resources

##### Mesh

To port the Product Catalog Apps to App Mesh, first you will need to create a mesh. You’ll also apply labels to the `prodcatalog-ns`namespace to affiliate your new mesh with it, and to enable automatic sidecar injection for pods within it. Also add the `gateway` label which we will use in the next section for setting up the `VirtualGateway`. Looking at the section of `mesh.yaml` shown below, you can see we’ve added the required labels to the `prodcatalog-ns` namespace and specified our mesh named `prodcatalog-mesh`.

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: prodcatalog-ns
  labels:
    mesh: prodcatalog-mesh
    gateway: ingress-gw
    appmesh.k8s.aws/sidecarInjectorWebhook: enabled
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: prodcatalog-mesh
spec:
  namespaceSelector:
    matchLabels:
      mesh: prodcatalog-mesh
---
```

##### VirtualNode

Kubernetes application objects that run within App Mesh must be defined as `VirtualNode`. This provides App Mesh an abstraction to objects such as Kubernetes Deployments and Services, and provides endpoints for communication and routing configuration. Looking at the `meshed_app.yaml`, below is the `frontend-node` service’s `VirtualNode` specification.

```
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: frontend-node
  namespace: prodcatalog-ns
spec:
  podSelector:
    matchLabels:
      app: frontend-node
      version: v1
  listeners:
    - portMapping:
        port: 9000
        protocol: http
  backends:
    - virtualService:
        virtualServiceRef:
          name: prodcatalog
    - virtualService:
        virtualServiceRef:
          name: prodsummary
  serviceDiscovery:
    dns:
      hostname: frontend-node.prodcatalog-ns.svc.cluster.local
  logging:
    accessLog:
      file:
        path: /dev/stdout
---
```

Note that it uses a `podSelector` to identify which `Pods` are members of this VirtualNode, as well as a pointer to the frontend-node `Service`.

##### VirtualService and VirtualRouter

There are also VirtualService and VirtualRouter specifications for each of the product catalog detail versions, establishing traffic routing to their respective endpoints. This is accomplished by adding `Route`s which point to the `proddetail-v1` virtual nodes. App Mesh also provides the `VirtualService` construct which allows you to specify a logical service path for application traffic. In this example, they send traffic to `VirtualRouters`, which then route traffic to the `VirtualNodes`. Looking at the `meshed_app.yaml`, below is the `proddetail` `VirtualService` and `VirtualRouter` which will route the traffic to version 1 of backend service `proddetail-v1`.

```
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: proddetail
  namespace: prodcatalog-ns
spec:
  awsName: proddetail.prodcatalog-ns.svc.cluster.local
  provider:
    virtualRouter:
      virtualRouterRef:
        name: proddetail-router
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  name: proddetail-router
  namespace: prodcatalog-ns
spec:
  listeners:
    - portMapping:
        port: 3000
        protocol: http
  routes:
    - name: proddetail-route
      httpRoute:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeRef:
                name: proddetail-v1
              weight: 100
---
```

With the basic constructs understood, it’s time to create the mesh and its resources.

# CREATE THE MESHED APPLICATION

#### Create Mesh Object

Configure namespace with App Mesh Labels and deploy Mesh Object

```bash
cd ~/environment/eks-app-mesh-polyglot-demo
kubectl apply -f deployment/mesh.yaml  

namespace/prodcatalog-ns configured 
mesh.appmesh.k8s.aws/prodcatalog-mesh created
```

Confirm the Mesh object and Namespace are created

```bash
kubectl describe namespace prodcatalog-ns

Name:         prodcatalog-ns
Labels:       appmesh.k8s.aws/sidecarInjectorWebhook=enabled
            gateway=ingress-gw
            mesh=prodcatalog-mesh
Annotations:  Status:  Active
kubectl describe mesh prodcatalog-mesh
Status:
Conditions:
    Last Transition Time:  2020-11-02T16:43:03Z
    Status:                True
    Type:                  MeshActive
```

#### Create App Mesh Resources for the services

```bash
kubectl apply -f deployment/meshed_app.yaml

virtualnode.appmesh.k8s.aws/prodcatalog created 
virtualservice.appmesh.k8s.aws/prodcatalog created 
virtualservice.appmesh.k8s.aws/proddetail created 
virtualrouter.appmesh.k8s.aws/proddetail-router created 
virtualrouter.appmesh.k8s.aws/prodcatalog-router created 
virtualnode.appmesh.k8s.aws/proddetail-v1 created 
virtualnode.appmesh.k8s.aws/frontend-node created 
virtualservice.appmesh.k8s.aws/frontend-node created 
```

Get all the Meshed resources for your application services, you should see below response.

```bash
kubectl get virtualnode,virtualservice,virtualrouter -n prodcatalog-ns

NAME                                        ARN                                                                                                     AGE
virtualnode.appmesh.k8s.aws/frontend-node   arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualNode/frontend-node_prodcatalog-ns   3m4s
virtualnode.appmesh.k8s.aws/prodcatalog     arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualNode/prodcatalog_prodcatalog-ns     35m
virtualnode.appmesh.k8s.aws/proddetail-v1   arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualNode/proddetail-v1_prodcatalog-ns   35m

NAME                                           ARN                                                                                                                          AGE
virtualservice.appmesh.k8s.aws/frontend-node   arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualService/frontend-node.prodcatalog-ns.svc.cluster.local   3m4s
virtualservice.appmesh.k8s.aws/prodcatalog     arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualService/prodcatalog.prodcatalog-ns.svc.cluster.local     35m
virtualservice.appmesh.k8s.aws/proddetail      arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualService/proddetail.prodcatalog-ns.svc.cluster.local      35m

NAME                                               ARN                                                                                                            AGE
virtualrouter.appmesh.k8s.aws/prodcatalog-router   arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualRouter/prodcatalog-router_prodcatalog-ns   35m
virtualrouter.appmesh.k8s.aws/proddetail-router    arn:aws:appmesh:us-west-2:$ACCOUNT_ID:mesh/prodcatalog-mesh/virtualRouter/proddetail-router_prodcatalog-ns    35m
```



#### Go to Console and check the App Mesh Resources information

[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/mesh_console.png)](https://www.eksworkshop.com/images/app_mesh_fargate/mesh_console.png)[![virtual nodes](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-3.png)](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-3.png)[![virtual services](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-1.png)](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-1.png)[![virtual routers](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-2.png)](https://www.eksworkshop.com/images/app_mesh_fargate/mesh-2.png)





# SIDECAR INJECTION

For a pod in your application to join a mesh, it must have an Envoy proxy container running sidecar within the pod. This establishes the data plane that AWS App Mesh controls. So we must run an Envoy container within each pod of the Product Catalog App deployment. For example:

[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/pcapp.png)](https://www.eksworkshop.com/images/app_mesh_fargate/pcapp.png)

This can be accomplished a few different ways:

- Before installing the application, you can modify the Product Catalog App `Deployment` container specs to include App Mesh sidecar containers and set a few required configuration elements and environment variables. When pods are deployed, they would run the sidecar.
- After installing the application, you can patch each `Deployment` to include the sidecar container specs. Upon applying this patch, the old pods would be torn down, and the new pods would come up with the sidecar.
- You can enable the AWS App Mesh Sidecar Injector in the meshed namespace, which watches for new pods to be created and automatically adds the sidecar container and required configuration to the pods as they are deployed.

In this tutorial, we will use the third option and enable automatic sidecar injection for our meshed pods. We have enabled automatic sidecar injection by adding label `Labels: appmesh.k8s.aws/sidecarInjectorWebhook=enabled` on the `prodcatalog-ns` namespace when we created the mesh resources in the previous lab, but this was done after initial pod creation. Currently, our pods each have one container running.

```bash
kubectl get pods -n prodcatalog-ns -o wide

NAME                                 READY   STATUS    RESTARTS   AGE   IP               NODE                                                   NOMINATED NODE   READINESS GATES
pod/frontend-node-77d64585d4-xxxx   1/1     Running   0          13h   192.168.X.6     ip-192-168-X-X.us-west-2.compute.internal           <none>           <none>
pod/prodcatalog-98f7c5f87-xxxxx      1/1     Running   0          13h   192.168.X.17   fargate-ip-192-168-X-X.us-west-2.compute.internal   <none>           <none>
pod/proddetail-5b558df99d-xxxxx      1/1     Running   0          18h   192.168.24.X   ip-192-168-X-X.us-west-2.compute.internal            <none>           <none>
```

To inject sidecar proxies for these pods, simply restart the deployments. The controller will handle the rest.

```bash
kubectl -n prodcatalog-ns rollout restart deployment prodcatalog

kubectl -n prodcatalog-ns rollout restart deployment proddetail 

kubectl -n prodcatalog-ns rollout restart deployment frontend-node
```

Get the Pod details. You should see 3 containers in each pod `main application container`, `envoy sidecar container` and `xray sidecar container`

It takes 4 to 6 minutes to restart the Fargate Pod

```bash
kubectl get pods -n prodcatalog-ns -o wide

NAME                                 READY   STATUS    RESTARTS   AGE   IP               NODE                                                   NOMINATED NODE   READINESS GATES
pod/frontend-node-77d64585d4-xxxx   3/3     Running   0          13h   192.168.X.6     ip-192-168-X-X.us-west-2.compute.internal           <none>           <none>
pod/prodcatalog-98f7c5f87-xxxxx      3/3     Running   0          13h   192.168.X.17   fargate-ip-192-168-X-X.us-west-2.compute.internal   <none>           <none>
pod/proddetail-5b558df99d-xxxxx      3/3     Running   0          18h   192.168.24.X   ip-192-168-X-X.us-west-2.compute.internal            <none>           <none>                                                                       3000/TCP       44h   app=proddetail,version=v1
```

#### Get Running containers from pod

We can see that there are two sidecar containers `envoy` and `xray-daemon` along with application container `frontend-node`

```bash
POD=$(kubectl -n prodcatalog-ns get pods -o jsonpath='{.items[0].metadata.name}')
kubectl -n prodcatalog-ns get pods ${POD} -o jsonpath='{.spec.containers[*].name}'; echo

frontend-node envoy xray-daemon
```



# TESTING THE APPLICATION

Now it’s time to test.[![App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/meshify.png)](https://www.eksworkshop.com/images/app_mesh_fargate/meshify.png)

To test if our ported Product Catalog App is working as expected, we’ll first exec into the `frontend-node` container.

```bash
export FE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=frontend-node -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${FE_POD_NAME} -c frontend-node bash
```

You will see a prompt from within the `frontend-node` container.

```
root@frontend-node-9d46cb55-XXX:/usr/src/app#
```

Test the configuration by issuing a curl request to the virtual service `prodcatalog` on port 5000, simulating what would happen if code running in the same pod made a request to the `prodcatalog` backend:

```bash
curl -v http://prodcatalog.prodcatalog-ns.svc.cluster.local:5000/products/    
```

Output should be similar to below. You can see that the request to the backend service `prodcatalog` is going via envoy proxy.

```
*   Trying 10.100.163.192... 
* TCP_NODELAY set 
* Connected to prodcatalog.prodcatalog-ns.svc.cluster.local (10.100.xx.yyy) port 5000 (#0)     
> GET /products/ HTTP/1.1     
> Host: prodcatalog.prodcatalog-ns.svc.cluster.local:5000    
> User-Agent: curl/7.52.1    
> Accept: */*    
>     
< HTTP/1.1 200 OK    
< content-type: application/json   
< content-length: 51    
< x-amzn-trace-id: Root=1-600925c6-e2c7bec92b824ddc9969d1b5    
< access-control-allow-origin: *   
< server: envoy    
< date: Thu, 21 Jan 2021 06:57:10 GMT   
< x-envoy-upstream-service-time: 19 
<  
{
    "products": {},
    "details": {
        "version": "1",
        "vendors": [
            "ABC.com"
        ]
    }
}
* Curl_http_done: called premature == 0  
* Connection #0 to host prodcatalog.prodcatalog-ns.svc.cluster.local left intact 
```

Exit from the `frontend-node` container. Now, To test the connectivity from the Fargate service `prodcatalog` to Nodegroup service `proddetail`, we’ll first exec into the `prodcatalog` container.

```bash
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${BE_POD_NAME} -c prodcatalog bash
```

You will see a prompt from within the `prodcatalog` container.

```
root@prodcatalog-98f7c5f87-xxxxx:/usr/src/app#
```

Test the configuration by issuing a curl request to the virtual service `proddetail` on port 3000, simulating what would happen if code running in the same pod made a request to the `proddetail` backend:

```bash
curl -v http://proddetail.prodcatalog-ns.svc.cluster.local:3000/catalogDetail 
```

You should see the below response. You can see that the request to the backend service `proddetail-v1` is going via envoy proxy. You can now exit from the `prodcatalog` exec bash.

```
.....
.....
< HTTP/1.1 200 OK    
< content-type: application/json   
< content-length: 51    
< x-amzn-trace-id: Root=1-600925c6-e2c7bec92b824ddc9969d1b5    
< access-control-allow-origin: *   
< server: envoy    
< date: Thu, 21 Jan 2021 06:57:10 GMT   
< x-envoy-upstream-service-time: 19 
....
....
{"version":"1","vendors":["ABC.com"]}
```

Congrats! You’ve migrated the initial architecture to provide the same functionality. Now lets expose the `frontend-node` to external users to access the UI using App Mesh Virtual Gateway.

# VIRTUALGATEWAY SETUP

A `VirtualGateway` allows resources that are outside of your mesh to communicate to resources that are inside of your mesh. Unlike a VirtualNode, which represents Envoy running with an application, a `VirtualGateway` represents Envoy deployed by itself.

External resources must be able to resolve a DNS name to an IP address assigned to the service or instance that runs Envoy. Envoy can then access all of the App Mesh configuration for resources that are inside of the mesh.

The configuration for handling the incoming requests at the VirtualGateway are specified using Gateway Routes. VirtualGateways are affiliated with a load balancer and allow you to configure ingress traffic rules using Routes, similar to VirtualRouter configuration.

[![Product Catalog App with App Mesh](https://www.eksworkshop.com/images/app_mesh_fargate/virtualgateway.png)](https://www.eksworkshop.com/images/app_mesh_fargate/virtualgateway.png)



# ADD VIRTUALGATEWAY

#### Adding App Mesh VirtualGateway

[![gateway](https://www.eksworkshop.com/images/app_mesh_fargate/meshify-vg.png)](https://www.eksworkshop.com/images/app_mesh_fargate/meshify-vg.png)Until now we have verified the communication between services is routed through envoy proxy, lets expose the frontend service `frontend-node` using AWS App Mesh VirtualGateway.

Create VirtualGateway components using the `virtual_gateway.yaml` as shown below. This will create the Kubernetes service as Type `LoadBalancer` and will use the AWS Network Load balancer for routing the external internet traffic.

```bash
kubectl apply -f deployment/virtual_gateway.yaml 

virtualgateway.appmesh.k8s.aws/ingress-gw created    
gatewayroute.appmesh.k8s.aws/gateway-route-frontend created 
service/ingress-gw created 
deployment.apps/ingress-gw created 
```

#### Get all the resources that are running in the namespace

You can see VirtualGateway components named as `ingress-gw` below:

```bash
kubectl get all  -n prodcatalog-ns -o wide | grep ingress

pod/ingress-gw-5fb995f6fd-45nnm      2/2     Running   0          35s     192.168.24.144    ip-192-168-21-156.us-west-2.compute.internal            <none>           <none>
service/ingress-gw      LoadBalancer   10.100.24.17     ad34ee9dea9944ed78e78d0578060ba6-869c67fd174d0f4d.elb.us-west-2.amazonaws.com   80:31569/TCP   35s   app=ingress-gw
deployment.apps/ingress-gw      1/1     1            1           35s   envoy           840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.15.1.0-prod        app=ingress-gw
replicaset.apps/ingress-gw-5fb995f6fd      1         1         1       35s     envoy           840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.15.1.0-prod        app=ingress-gw,pod-template-hash=5fb995f6fd
virtualgateway.appmesh.k8s.aws/ingress-gw   arn:aws:appmesh:us-west-2:405710966773:mesh/prodcatalog-mesh/virtualGateway/ingress-gw_prodcatalog-ns   35s
gatewayroute.appmesh.k8s.aws/gateway-route-frontend   arn:aws:appmesh:us-west-2:405710966773:mesh/prodcatalog-mesh/virtualGateway/ingress-gw_prodcatalog-ns/gatewayRoute/gateway-route-frontend_prodcatalog-ns   35s
```

Log into console and navigate to AWS App Mesh -> Click on `prodcatalog-mesh` -> Click on `Virtual gateways`, you should see below page.[![vgateway](https://www.eksworkshop.com/images/app_mesh_fargate/vg.png)](https://www.eksworkshop.com/images/app_mesh_fargate/vg.png)

# TESTING VIRTUAL GATEWAY

Now it’s time to test. Get the load balancer endpoint that Virtual Gateway is exposed.

It takes 3 to 5 minutes to set up the Load Balancer. You can go to Console and Go to Load Balancer and check if the state is `Active`. Do not proceed to next step until Load Balancer is `Active`.

```bash
export LB_NAME=$(kubectl get svc ingress-gw -n prodcatalog-ns -o jsonpath="{.status.loadBalancer.ingress[*].hostname}") 
 curl -v --silent ${LB_NAME} | grep x-envoy
echo $LB_NAME
```

Check if the request to the Ingress Gateway is going from envoy by curl to the above Loadbalancer url

```
> GET / HTTP/1.1
> Host: db13be460b8648c4bXXXf.elb.us-west-2.amazonaws.com
> User-Agent: curl/7.54.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
< content-length: 3783
< x-amzn-trace-id: Root=1-5ff4c10e-71cca9a19486406b80eaa475
< server: envoy
< date: Tue, 05 Jan 2021 19:42:06 GMT
< x-envoy-upstream-service-time: 3
< server: envoy
{ [1079 bytes data]
* Connection #0 to host db13be460b8648c4bXXXf.elb.us-west-2.amazonaws.com left intact

workshop:~/environment $ echo $LB_NAME

db13be460b8648c4bXXXf.elb.us-west-2.amazonaws.com
```



Copy paste the above Loadbalancer endpoint in your browser and you should see the frontend application loaded as below.[![fronteend](https://www.eksworkshop.com/images/app_mesh_fargate/ui1.png)](https://www.eksworkshop.com/images/app_mesh_fargate/ui1.png)

Add Product e.g. `Table` with ID as `1` to Product Catalog and click **Add** button.[![post rquest](https://www.eksworkshop.com/images/app_mesh_fargate/ui2.png)](https://www.eksworkshop.com/images/app_mesh_fargate/ui2.png)

You should see the new product `Table` added in the Product Catalog table. You can also see the Catalog Detail about Vendor information has been fetched from `proddetail` backend service.[![pos2 reauest](https://www.eksworkshop.com/images/app_mesh_fargate/ui3.png)](https://www.eksworkshop.com/images/app_mesh_fargate/ui3.png)

Congratulations on exposing the Product Catalog Application via App Mesh Virtual Gateway!

