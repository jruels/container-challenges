# GITOPS WITH WEAVE FLUX

[GitOps](https://www.weave.works/technologies/gitops/), a term coined by Weaveworks, is a way to do continuous delivery. Git is used as a single source of truth for deploying into your cluster. This is easy for a development teams as they are already familiar with `git` and do not need to know other tools. [Weave Flux](https://www.weave.works/oss/flux/) is a tool that runs in your Kubernetes cluster and implements changes based on monitoring Git and image repositories.

In this lab, we will create a Docker image build pipeline using AWS CodePipeline for a sample application in a GitHub repository. We will then commit Kubernetes manifests to GitHub and monitor Weave Flux managing the deployment.

Below is a diagram of what will be created:

[![GitOps Workflow](https://www.eksworkshop.com/images/weave_flux/gitops_workflow.png)](https://www.eksworkshop.com/images/weave_flux/gitops_workflow.png)



### PREREQUISITES

AWS CodePipeline and AWS CodeBuild both need AWS Identity and Access Management (IAM) service roles to create a Docker image build pipeline.

In this step, we are going to create an IAM role and add an inline policy that we will use in the CodeBuild stage to interact with the EKS cluster via `kubectl`.

Create the bucket and roles:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
aws s3 mb s3://eksworkshop-${ACCOUNT_ID}-codepipeline-artifacts

cd ~/environment

wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cpAssumeRolePolicyDocument.json

aws iam create-role --role-name eksworkshop-CodePipelineServiceRole --assume-role-policy-document file://cpAssumeRolePolicyDocument.json 

wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cpPolicyDocument.json

aws iam put-role-policy --role-name eksworkshop-CodePipelineServiceRole --policy-name codepipeline-access --policy-document file://cpPolicyDocument.json

wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cbAssumeRolePolicyDocument.json

aws iam create-role --role-name eksworkshop-CodeBuildServiceRole --assume-role-policy-document file://cbAssumeRolePolicyDocument.json 

wget https://eksworkshop.com/intermediate/260_weave_flux/iam.files/cbPolicyDocument.json

aws iam put-role-policy --role-name eksworkshop-CodeBuildServiceRole --policy-name codebuild-access --policy-document file://cbPolicyDocument.json
```



### GITHUB SETUP

We are going to create 2 GitHub repositories. One will be used for a sample application that will trigger a Docker image build. Another will be used to hold Kubernetes manifests that Weave Flux deploys into the cluster. Note this is a pull-based method compared to other continuous deployment tools that push to Kubernetes.

Create the sample application repository by clicking [here](https://github.com/new).

Fill in the form with the repository name, description, and check to initialize the repository with a ``README as shown below and click **Create repository**.

[![Create Sample App Repository](https://www.eksworkshop.com/images/weave_flux/github_create_sample_app.png)](https://www.eksworkshop.com/images/weave_flux/github_create_sample_app.png)

Repeat this process to create the Kubernetes manifests repositories by clicking [here](https://github.com/new). Fill in the form as shown below and click **Create repository**.

[![Create Kubernetes Manifest Repository](https://www.eksworkshop.com/images/weave_flux/github_create_k8s.png)](https://www.eksworkshop.com/images/weave_flux/github_create_k8s.png)

The next step is to create a personal access token that will allow CodePipeline to receive callbacks from GitHub.
Once created, an access token can be stored in a secure enclave and reused, so this step is only required during the first run or when you need to generate new keys.

Open up the [New personal access page](https://github.com/settings/tokens/new) in GitHub.

You may be prompted to enter your GitHub password

Enter a value for **Token description**, check the **repo** permission scope and scroll down and click the **Generate token** button

[![Generate New](https://www.eksworkshop.com/images/weave_flux/github_token_name.png)](https://www.eksworkshop.com/images/weave_flux/github_token_name.png)

Copy the **personal access token** and save it in a secure place for the next step

[![Generate New](https://www.eksworkshop.com/images/weave_flux/github_copy_access.png)](https://www.eksworkshop.com/images/weave_flux/github_copy_access.png)

We will need to revisit GitHub one more time once we provision Weave Flux to enable Weave to control repositories. However, at this time you can move on.



### INSTALL WEAVE FLUX

Now we will use Helm to install Weave Flux into our cluster and enable it to interact with our Kubernetes configuration GitHub repo.

First, install the Flux Custom Resource Definition:

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```



> In the following steps, your Git user name will be required. Without this information, the resulting pipeline will not function as expected. Set this as an environment variable to reuse in the next commands:

```bash
YOURUSER=yourgitusername
```

First, create the flux Kubernetes namespace

```
kubectl create namespace flux
```

Next, add the Flux chart repository to Helm and install Flux.

Update the Git URL below to match your user name and Kubernetes configuration manifest repository.

```
helm repo add fluxcd https://charts.fluxcd.io

helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:${YOURUSER}/k8s-config \
--set git.branch=main \
--namespace flux

helm upgrade -i helm-operator fluxcd/helm-operator \
--set helm.versions=v3 \
--set git.ssh.secretName=flux-git-deploy \
--set git.branch=main \
--namespace flux
```

Watch the install and confirm everything starts. There should be 3 pods.

```
kubectl get pods -n flux
```

Install fluxctl in order to get the SSH key to allow GitHub write access. This allows Flux to keep the configuration in GitHub in sync with the configuration deployed in the cluster.

```
sudo wget -O /usr/local/bin/fluxctl $(curl https://api.github.com/repos/fluxcd/flux/releases/latest | jq -r ".assets[] | select(.name | test(\"linux_amd64\")) | .browser_download_url")
sudo chmod 755 /usr/local/bin/fluxctl

fluxctl version
fluxctl identity --k8s-fwd-ns flux
```

Copy the provided key and add that as a deploy key in the GitHub repository.

- In GitHub, select your k8s-config GitHub repo. Go to **Settings** and click **Deploy Keys**. Alternatively, you can go by direct URL by replacing your user name in this URL: **github.com/YOURUSER/k8s-config/settings/keys**.
- Click on **Add Deploy Key**
- Name: **Flux Deploy Key**
- Paste the key output from fluxctl
- Click **Allow Write Access**. This allows Flux to keep the repo in sync with the real state of the cluster
- Click **Add Key**

Now Flux is configured and should be ready to pull configuration.



### CREATE IMAGE WITH CODEPIPELINE

Now we are going to create the AWS CodePipeline using AWS CloudFormation. This pipeline will be used to build a Docker image from your GitHub source repo `eks-example`. Note that this does not deploy the image. Weave Flux will handle that.

CloudFormation is an infrastructure as code (IaC) tool which provides a common language for you to describe and provision all the infrastructure resources in your cloud environment. CloudFormation allows you to use a simple text file to model and provision, in an automated and secure manner, all the resources needed for your applications across all regions and accounts.

Each EKS deployment/service should have its own CodePipeline and be located in an isolated source repository.

Click the **Launch** button to create the CloudFormation stack in the AWS Management Console.

| Launch template    |                                                              |                                                              |
| :----------------- | :----------------------------------------------------------: | :----------------------------------------------------------: |
| CodePipeline & EKS | [ Launch](https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?stackName=image-codepipeline&templateURL=https://s3.amazonaws.com/eksworkshop.com/templates/main/weave_flux_pipeline.cfn.yml) | [ Download](https://s3.amazonaws.com/eksworkshop.com/templates/main/weave_flux_pipeline.cfn.yml) |

After the console is open, replace YOURUSER with your GitHub username, personal access token (created in previous step) and then click the “Create stack” button located at the bottom of the page.

![CloudFormation Stack](https://www.eksworkshop.com/images/weave_flux/cloudformation_stack.png)

Wait for the status to change from “CREATE_IN_PROGRESS” to **CREATE_COMPLETE** before moving on to the next step.

[![CloudFormation Stack](https://www.eksworkshop.com/images/weave_flux/cloudformation_stack_creating.png)](https://www.eksworkshop.com/images/weave_flux/cloudformation_stack_creating.png)

Open [CodePipeline in the Management Console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines). You will see a CodePipeline that starts with **image-codepipeline**. Click this link to view the details.

If you receive a permissions error similar to **User x is not authorized to perform: codepipeline:ListPipelines…** upon clicking the above link, the CodePipeline console may have opened up in the wrong region. To correct this, from the **Region** dropdown in the console, choose the region you provisioned the workshop in.

Currently, the image build is likely failed because we have no code in our repository. We will add a sample application to our GitHub repo `eks-example`. Clone the repo substituting your GitHub user name.

```
git clone https://github.com/${YOURUSER}/eks-example.git
cd eks-example
```

Next create a base README file, a source directory, and download a sample nginx configuration (hello.conf), home page (index.html), and Dockerfile.

```
echo "# eks-example" > README.md
mkdir src
wget -O src/hello.conf https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/hello.conf
wget -O src/index.html https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/index.html
wget https://raw.githubusercontent.com/aws-samples/eks-workshop/main/content/intermediate/260_weave_flux/app.files/Dockerfile
```

Now that we have a simple hello world app, commit the changes to start the image build pipeline.

**NOTE: User the Personal Access Token created earlier as the password when executing `git push`

```
git add .
git commit -am "Initial commit"
git push 
```

In the [CodePipeline console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) go to the details page for the specific CodePipeline. You can see the status along with links to the change and build details.

[![CodePipeline Details](https://www.eksworkshop.com/images/weave_flux/codepipeline_details.png)](https://www.eksworkshop.com/images/weave_flux/codepipeline_details.png)

If you click on the “details” link in the build/deploy stage, you can see the output from the CodeBuild process.

To verify the image is built, go to the [Amazon ECR console](https://console.aws.amazon.com/ecr/repositories) and look for your **eks-example** image repository.



### DEPLOY FROM MANIFESTS

Now we are ready to use Weave Flux to deploy the hello world application into our Amazon EKS cluster. To do this we will clone our GitHub config repository (k8s-config) and then commit Kubernetes manifests to deploy.

```
cd ..
git clone https://github.com/${YOURUSER}/k8s-config.git     
cd k8s-config
mkdir charts namespaces releases workloads
```

Create a namespace Kubernetes manifest.

```
cat << EOF > namespaces/eks-example.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    name: eks-example
  name: eks-example
EOF
```

Create a deployment Kubernetes manifest.

Update the image below to point to your ECR repository and image tag (**Do NOT use latest**). You can find your Image URI from the [Amazon ECR Console](https://console.aws.amazon.com/ecr/repositories/eks-example/). Replace YOURACCOUNT and YOURTAG)

```
cat << EOF > workloads/eks-example-dep.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-example
  namespace: eks-example
  labels:
    app: eks-example
  annotations:
    # Container Image Automated Updates
    flux.weave.works/automated: "true"
    # do not apply this manifest on the cluster
    #flux.weave.works/ignore: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eks-example
  template:
    metadata:
      labels:
        app: eks-example
    spec:
      containers:
      - name: eks-example
        image: YOURACCOUNT.dkr.ecr.us-east-1.amazonaws.com/eks-example:YOURTAG
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http
        readinessProbe:
          httpGet:
            path: /
            port: http
EOF
```

Above you see 2 Kubernetes annotations for Flux.

- `flux.weave.works/automated` tells Flux whether the container image should be automatically updated.
- `flux.weave.works/ignore` is commented out, but could be used to tell Flux to temporarily ignore the deployment.

Finally, create a service manifest to enable a load balancer to be created.

```
cat << EOF > workloads/eks-example-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: eks-example
  namespace: eks-example
  labels:
    app: eks-example
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: eks-example
EOF
```

Now commit the changes and push to your repository.

```
git add . 
git commit -am "eks-example-deployment"
git push 
```

Check the logs of your Flux pod. It will pull the config from the `k8s-config` repository every 5 minutes. Ensure you replace the pod name below with the name in your deployment.

```
kubectl get pods -n flux

kubectl logs flux-5bd7fb6bb6-4sc78 -n flux
```

Now get the URL for the load balancer (LoadBalancer Ingress) and connect via your browser (this may take a couple of minutes for DNS).

```
kubectl describe service eks-example -n eks-example
```

Make a change to the eks-example source code and push a new change.

```
cd ../eks-example
vi src/index.html
   # Change the <title> AND <h> to Hello World Version 2

git commit -am "v2 Updating home page"
git push
```

Now you can watch in the [CodePipeline console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) for the new image build to complete. This will take a couple of minutes. Once complete, you will see a new image land in your [Amazon ECR repository](https://console.aws.amazon.com/ecr/repositories/eks-example/). Monitor the **kubectl logs** for the Flux pod and you should see it update the configuration within five minutes.

Verify the web page has been updated by refreshing the page in your browser.



### Cleanup 

Congratulations on completing the GitOps with Weave Flux module.

First, delete all images from the [Amazon ECR Repository](https://console.aws.amazon.com/ecr/repositories).

Next, go to the [CloudFormation Console](https://console.aws.amazon.com/cloudformation/) and delete the stack used to deploy the image build CodePipeline

Now, delete Weave Flux and your load-balanced services

```bash
helm uninstall helm-operator --namespace flux
helm uninstall flux --namespace flux
kubectl delete namespace flux 
kubectl delete crd helmreleases.helm.fluxcd.io

helm uninstall mywebserver -n nginx
kubectl delete namespace nginx
kubectl delete svc eks-example -n eks-example
kubectl delete deployment eks-example -n eks-example
kubectl delete namespace eks-example
```

Optionally go to GitHub and delete your `k8s-config` and `eks-example` repositories.

Remove IAM roles you previously created

```
aws iam delete-role-policy --role-name eksworkshop-CodePipelineServiceRole --policy-name codepipeline-access 
aws iam delete-role --role-name eksworkshop-CodePipelineServiceRole
aws iam delete-role-policy --role-name eksworkshop-CodeBuildServiceRole --policy-name codebuild-access 
aws iam delete-role --role-name eksworkshop-CodeBuildServiceRole
```

Remove the artifact bucket you previously created

```
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
aws s3 rb s3://eksworkshop-${ACCOUNT_ID}-codepipeline-artifacts --force
```

### Congrats! 

You have completed the lab. 



