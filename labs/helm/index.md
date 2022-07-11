# INSTALL HELM

## Install the Helm CLI

Before we can get started configuring Helm, we’ll need to first install the command line tools that you will interact with. To do this, run the following:

```sh
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

We can verify the version

```sh
helm version --short
```

Let’s configure our first Chart repository. Chart repositories are similar to APT or yum repositories that you might be familiar with on Linux, or Taps for Homebrew on macOS.

Download the `bitnami` repository so we have something to start with:

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Once this is installed, we will be able to list the charts you can install:

```sh
helm search repo bitnami
```

The output will show many charts that are maintained by Bitnami.



Finally, let’s configure Bash completion for the `helm` command:

```sh
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```



### SEARCH CHART REPOSITORIES

Now that our repository Chart list has been updated, we can [search for Charts](https://helm.sh/docs/helm/helm_search/).

To list all Charts:

```sh
helm search repo
```

That should output something similar to:

```
NAME                                            CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/airflow                                 12.5.13         2.3.3           Apache Airflow is a tool to express and execute...
bitnami/apache                                  9.1.13          2.4.54          Apache HTTP Server is an open-source HTTP serve...
bitnami/argo-cd                                 3.4.5           2.4.4           Argo CD is a continuous delivery tool for Kuber...
bitnami/argo-workflows                          2.3.5           3.3.8           Argo Workflows is meant to orchestrate Kubernet...
bitnami/aspnet-core                             3.4.11          6.0.6           ASP.NET Core is an open-source framework for we...
bitnami/cassandra                               9.2.7           4.0.4           Apache Cassandra is an open source distributed ...
```

You can see from the output that it dumped the list of all Charts we have added. In some cases that may be useful, but an even more useful search would involve a keyword argument. So next, we’ll search just for `nginx`:

```sh
helm search repo nginx
```

That results in:

```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/nginx                           13.1.0          1.23.0          NGINX Open Source is a web server that can be a...
bitnami/nginx-ingress-controller        9.2.18          1.2.1           NGINX Ingress Controller is an Ingress controll...
bitnami/nginx-intel                     2.0.12          0.4.7           NGINX Open Source for Intel is a lightweight se...
bitnami/kong                            5.0.2           2.7.0           Kong is a scalable, open source API layer (aka ...
```

This new list of Charts is specific to `nginx` because we passed the **nginx** argument to the `helm search repo` command.



### INSTALL BITNAMI/NGINX

Installing the Bitnami standalone `nginx` web server Chart involves us using the [helm install](https://helm.sh/docs/helm/helm_install/) command.

A Helm Chart can be installed multiple times inside a Kubernetes cluster. This is because each installation of a Chart can be customized to suit a different purpose.

For this reason, you must supply a unique name for the installation, or ask Helm to generate a name for you.

#### Challenge:

**How can you use Helm to deploy the bitnami/nginx chart?**

**HINT:** Use the `helm` utility to `install` the `bitnami/nginx` chart and specify the name `mywebserver` for the Kubernetes deployment. Consult the [helm install](https://helm.sh/docs/intro/quickstart/#install-an-example-chart) documentation or run the `helm install --help` command to figure out the syntax.



If you run into issues or have questions please reach out to your instructor. 



Once you run this command, the output will contain the information about the deployment status, revision, namespace, etc, similar to:

```
NAME: mywebserver
LAST DEPLOYED: Thu Jul 15 13:52:34 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

NGINX can be accessed through the following DNS name from within your cluster:

    mywebserver-nginx.default.svc.cluster.local (port 80)

To access NGINX from outside the cluster, follow the steps below:

1. Get the NGINX URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w mywebserver-nginx'

    export SERVICE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].port}" services mywebserver-nginx)
    export SERVICE_IP=$(kubectl get svc --namespace default mywebserver-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "http://${SERVICE_IP}:${SERVICE_PORT}"
```

In order to review the underlying Kubernetes services, pods, and deployments, run:

```sh
kubectl get svc,po,deploy
```

In the following `kubectl` command examples, it may take a minute or two for each of these objects' `DESIRED` and `CURRENT` values to match; if they don’t match on the first try, wait a few seconds, and run the command again to check the status.

The first object shown in this output is a Deployment. A Deployment object manages rollouts (and rollbacks) of different versions of an application.

You can inspect this Deployment object in more detail by running the following command:

```
kubectl describe deployment mywebserver
```

The next object shown created by the Chart is a Pod. A Pod is a group of one or more containers.

To verify the Pod object was successfully deployed, we can run the following command:

```
kubectl get pods -l app.kubernetes.io/name=nginx
```

And you should see output similar to:

```
NAME                                 READY     STATUS    RESTARTS   AGE
mywebserver-nginx-85985c8466-tczst   1/1       Running   0          10s
```

The third object that this Chart creates for us is a Service. A Service enables us to contact this nginx web server from the Internet, via an Elastic Load Balancer (ELB).

To get the complete URL of this Service, run:

```
kubectl get service mywebserver-nginx
```

That should output something similar to:

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP
mywebserver-nginx   LoadBalancer   10.100.223.99   abc123.amazonaws.com
```

Copy the value for `EXTERNAL-IP`, open a new tab in your web browser, and paste it in.

It may take a couple of minutes for the ELB and its associated DNS name to become available; if you get an error, wait one minute, and hit reload.

When the Service does come online, you should see a welcome message similar to:

[![Helm Logo](https://www.eksworkshop.com/images/helm-nginx/welcome_to_nginx.png)](https://www.eksworkshop.com/images/helm-nginx/welcome_to_nginx.png)

### Congrats!

Congratulations! You’ve now successfully deployed the nginx standalone web server to your EKS cluster!



### CLEAN UP

To remove all the objects that the Helm Chart created, we can use [Helm uninstall](https://helm.sh/docs/helm/helm_uninstall/).

Before we uninstall our application, we can verify what we have running via the [Helm list](https://helm.sh/docs/helm/helm_list/) command:

```sh
helm list
```

You should see output similar to below, which shows that mywebserver is installed:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
mywebserver     default         1               2021-07-15 13:52:34.563653342 +0000 UTC deployed        nginx-9.3.7     1.21.1   
```



It was a lot of fun; we had some great times sending HTTP back and forth, but now it's time to uninstall this deployment. To uninstall:

```sh
helm uninstall mywebserver
```

And you should be met with the output:

```
release "mywebserver" uninstalled
```



Use `kubectl` to also demonstrate that our pods and service are no longer available:

```sh
kubectl get pods -l app.kubernetes.io/name=nginx
kubectl get service mywebserver-nginx
```

With that, cleanup is complete.



### Deploy Example Microservices Using Helm

Now we will demonstrate how to deploy microservices using a custom Helm Chart, instead of doing everything manually using `kubectl`.

Helm charts have a structure similar to:

```
/eksdemo
├── charts/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

We’ll follow this template, and create a new chart called **eksdemo** with the following commands:

```sh
cd ~/environment
helm create eksdemo
cd eksdemo
```



### CUSTOMIZE DEFAULTS

If you look in the newly created **eksdemo** directory, you’ll see several files and directories.

The table below outlines the purpose of each component in the Helm chart structure.

| File or Directory            | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| charts/                      | Sub-charts that the chart depends on                         |
| Chart.yaml                   | Information about your chart                                 |
| values.yaml                  | The default values for your templates                        |
| template/                    | The template files                                           |
| template/deployment.yaml     | Basic manifest for creating Kubernetes Deployment objects    |
| template/_helpers.tpl        | Used to define Go template helpers                           |
| template/hpa.yaml            | Basic manifest for creating Kubernetes Horizontal Pod Autoscaler objects |
| template/ingress.yaml        | Basic manifest for creating Kubernetes Ingress objects       |
| template/NOTES.txt           | A plain text file to give users detailed information about how to use the newly installed chart |
| template/serviceaccount.yaml | Basic manifest for creating Kubernetes ServiceAccount objects |
| template/service.yaml        | Basic manifest for creating Kubernetes Service objects       |
| tests/                       | Directory of Test files                                      |
| tests/test-connections.yaml  | Tests that validate that your chart works as expected when it is installed |

We’re actually going to create our own files, so we’ll delete these boilerplate files

```sh
rm -rf ~/environment/eksdemo/templates/
rm ~/environment/eksdemo/Chart.yaml
rm ~/environment/eksdemo/values.yaml
```

Run the following code block to create a new Chart.yaml file which will describe the chart

```sh
cat <<EoF > ~/environment/eksdemo/Chart.yaml
apiVersion: v2
name: eksdemo
description: A Helm chart for EKS Workshop Microservices application
version: 0.1.0
appVersion: 1.0
EoF
```

Next we’ll copy the manifest files for each of our microservices into the templates directory as *servicename*.yaml

```sh
#create subfolders for each template type
mkdir -p ~/environment/eksdemo/templates/deployment
mkdir -p ~/environment/eksdemo/templates/service

# Copy and rename frontend manifests
cp ~/environment/ecsdemo-frontend/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/frontend.yaml
cp ~/environment/ecsdemo-frontend/kubernetes/service.yaml ~/environment/eksdemo/templates/service/frontend.yaml

# Copy and rename crystal manifests
cp ~/environment/ecsdemo-crystal/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/crystal.yaml
cp ~/environment/ecsdemo-crystal/kubernetes/service.yaml ~/environment/eksdemo/templates/service/crystal.yaml

# Copy and rename nodejs manifests
cp ~/environment/ecsdemo-nodejs/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/nodejs.yaml
cp ~/environment/ecsdemo-nodejs/kubernetes/service.yaml ~/environment/eksdemo/templates/service/nodejs.yaml
```

All files in the templates directory are sent through the template engine. These are currently plain YAML files that would be sent to Kubernetes as-is.

## Replace hard-coded values with template directives

Let’s replace some of the values with `template directives` to enable more customization by removing hard-coded values.

Open ~/environment/eksdemo/templates/deployment/frontend.yaml in your Cloud9 editor.

The following steps should be completed separately for **frontend.yaml**, **crystal.yaml**, and **nodejs.yaml**.

Under `spec`, find **replicas: 1** and replace with the following:

```yaml
replicas: {{ .Values.replicas }}
```

Under `spec.template.spec.containers.image`, replace the image with the correct template value from the table below:

| Filename      | Value                                                       |
| :------------ | :---------------------------------------------------------- |
| frontend.yaml | - image: {{ .Values.frontend.image }}:{{ .Values.version }} |
| crystal.yaml  | - image: {{ .Values.crystal.image }}:{{ .Values.version }}  |
| nodejs.yaml   | - image: {{ .Values.nodejs.image }}:{{ .Values.version }}   |

#### Create a values.yaml file with our template defaults

Run the following code block to populate our `template directives` with default values.

```sh
cat <<EoF > ~/environment/eksdemo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
replicas: 3
version: 'latest'

# Service Specific Values
nodejs:
  image: brentley/ecsdemo-nodejs
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
EoF
```



### DEPLOY THE EKSDEMO CHART

#### Use the dry-run flag to test our templates

To test the syntax and validity of the Chart without actually deploying it, we’ll use the `--dry-run` flag.

The following command will build and output the rendered templates without installing the Chart:

```sh
helm install --debug --dry-run workshop ~/environment/eksdemo
```

Confirm that the values created by the template look correct.

#### Deploy the chart

Now that we have tested our template, let’s install it.

```sh
helm install workshop ~/environment/eksdemo
```

Similar to what we saw previously in the nginx Helm Chart example, an output of the command will contain the information about the deployment status, revision, namespace, etc, similar to:

```
NAME: workshop
LAST DEPLOYED: Sat Jul 17 08:47:32 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

In order to review the underlying services, pods, and deployments, run:

```sh
kubectl get svc,po,deploy
```



### TEST THE SERVICE

To test the service our eksdemo Chart created, we’ll need to get the name of the ELB endpoint that was generated when we deployed the Chart:

```sh
kubectl get svc ecsdemo-frontend -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"; echo
```

Copy that address, and paste it into a new tab in your browser. You should see something similar to:

[![Example Service](https://www.eksworkshop.com/images/helm_micro/micro_example.png)



### ROLLING BACK

Mistakes will happen during deployment, and when they do, Helm makes it easy to undo, or “roll back” to the previously deployed version.

#### Update the demo application chart with a breaking change

Open **values.yaml** and modify the image name under `nodejs.image` to **brentley/ecsdemo-nodejs-non-existing**. This image does not exist, so this will break our deployment.

Deploy the updated demo application chart:

```sh
helm upgrade workshop ~/environment/eksdemo
```

The rolling upgrade will begin by creating a new nodejs pod with the new image. The new `ecsdemo-nodejs` Pod should fail to pull the non-existing image. Run `kubectl get pods` to see the `ImagePullBackOff` error.

```sh
kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
ecsdemo-crystal-56976b4dfd-9f2rf    1/1     Running            0          2m10s
ecsdemo-frontend-7f5ddc5485-8vqck   1/1     Running            0          2m10s
ecsdemo-nodejs-56487f6c95-mv5xv     0/1     ImagePullBackOff   0          6s
ecsdemo-nodejs-58977c4597-r6hvj     1/1     Running            0          2m10s
```

Run `helm status workshop` to verify the `LAST DEPLOYED` timestamp.

```sh
helm status workshop
NAME: workshop
LAST DEPLOYED: Fri Jul 16 13:53:22 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
...
```

This should correspond to the last entry on `helm history workshop`

```sh
helm history workshop
```



#### Rollback the failed upgrade

Now we are going to roll back the application to the previous working release revision.

First, list Helm release revisions:

```sh
helm history workshop
```

Then, rollback to the previous application revision (can rollback to any revision too):

```sh
# rollback to the 1st revision
helm rollback workshop 1
```

Validate `workshop` release status and you will see a new revision that is based on the rollback.

```sh
helm status workshop
NAME: workshop
LAST DEPLOYED: Fri Jul 16 13:55:27 2021
NAMESPACE: default
STATUS: deployed
REVISION: 3
TEST SUITE: None
```

Verify that the error is gone

```sh
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
ecsdemo-crystal-56976b4dfd-9f2rf    1/1     Running   0          6m
ecsdemo-frontend-7f5ddc5485-8vqck   1/1     Running   0          6m
ecsdemo-nodejs-58977c4597-r6hvj     1/1     Running   0          6m
```



### CLEANUP

To delete the workshop release, run:

```sh
helm uninstall workshop
```