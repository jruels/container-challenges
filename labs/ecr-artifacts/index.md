# Using Elastic Container Registry

In this lab, you will: 

- Create an ECR repository
- Build a Docker image and push it to ECR 
- Run a Docker container from the image
- Create a Helm chart 
- Package the chart
- Push it to ECR
- Install the Helm chart from ECR.

The Elastic Container Registry can be used for storing container images and Helm Charts. 



### Create a Docker Repository

Run the following to create a Docker Repository 

```bash
aws ecr create-repository \
        --repository-name eks-workshop \
        --image-scanning-configuration scanOnPush=true \
        --region ${AWS_REGION}
```



Confirm the repository was successfully created 

```
aws ecr describe-repositories
```



To use the repository, you must authenticate. Amazon makes this easy by providing the `get-login-password` function. 

First, get your account ID:

```
aws sts get-caller-identity --query "Account" --output text
```



Run the following command, replacing {YOUR_ACCOUNT_ID} with the output from above:

```bash
aws ecr get-login-password \
        --region ${AWS_REGION} | docker login \
        --username AWS \
        --password-stdin {YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```



If it was successful, you will see output similar to: 

```
WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```



### Build a container image

Clone the GitHub repository: 

```
git clone https://github.com/jruels/docker-go-hello-world.git
```



Enter the directory and replace the `Dockerfile` with the following:

```dockerfile
FROM golang:1.9-alpine as builder

COPY main.go /go/src/github.com/jruels/docker-go-hello-world/

RUN cd /go/src/github.com/jruels/docker-go-hello-world && \
    go get && \
    CGO_ENABLED=0 GOOS=linux go build -a -o /go/bin/hello main.go

CMD ["/go/bin/hello"]

FROM alpine:latest
COPY --from=builder /go/bin/hello /usr/local/bin
CMD ["/usr/local/bin/hello"]
```



Run the `build` command, replacing {YOUR_ACCOUNT_ID}

```bash
docker build -t {YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-workshop:1.0 .
```



If the build is successful, the output will be similar to: 

```
Sending build context to Docker daemon  91.14kB
Step 1/7 : FROM golang:1.9-alpine as builder
 ---> b0260be938c6
 ..snip
 
Successfully built 14752e2084b5
Successfully tagged 477729760828.dkr.ecr.us-east-2.amazonaws.com/eks-workshop:1.0
```



To confirm the image was created, run `docker images`

```
REPOSITORY                                                  TAG          IMAGE ID       CREATED              SIZE
477729760828.dkr.ecr.us-east-2.amazonaws.com/eks-workshop   1.0          14752e2084b5   About a minute ago   7.39MB
```



### Push container image to ECR

Run the following to push the image to ECR

```
docker push {YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-workshop:1.0 
```



Successful output: 

```
The push refers to repository [477729760828.dkr.ecr.us-east-2.amazonaws.com/eks-workshop]
b7ca5eb6a0bc: Pushed 
24302eb7d908: Layer already exists 
2.0: digest: sha256:ad08d89e447b42a1c48fd2badd30d236c0986c0c35cf4a83ea8e353cd8552d3d size: 738
```



### Run container from image

Now that you've pushed an image to ECR use `docker run` to launch it. 

```
docker run ${YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-workshop:1.0
```

The container image is downloaded from ECR, and a container is run, which outputs `hello world!`



### Create a Helm chart 

This creates a simple demonstration chart. 

```
helm create helm-test-chart
```

Package the test chart

```
helm package helm-test-chart
```

You will see output saying the chart was saved to your current directory. 

### Create a Helm repository in ECR

AWS ECR supports artifacts, including container images and Helm charts.  These steps show how to push a Helm chart to an ECR repository. 

Create a Helm repository 

```bash
aws ecr create-repository \
    --repository-name helm-test-chart \
    --region ${AWS_REGION}
```



### Log into the Helm repository 

```
aws ecr get-login-password --region ${AWS_REGION} | helm registry login --username AWS --password-stdin {YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

### Push the Helm chart to the ECR repository

Now that we've created, and packaged a chart let's push it up to our ECR repository.

```
helm push helm-test-chart-0.1.0.tgz oci://{YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```



Confirm the Helm chart was pushed to the registry and has the `artifactMediaType` defined as `application/vnd.cncf.helm.config.v1+json`



### Deploy Helm chart 

Now that you've created a test chart and pushed it to ECR it's time to deploy the application. 

```
helm install ecr-chart-demo oci://{YOUR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/helm-test-chart --version=0.1.0
```



### Cleanup

After confirming the chart deployed successfully clean everything up. 

```
helm uninstall ecr-chart-demo
```

