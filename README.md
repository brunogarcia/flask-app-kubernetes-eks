# Deploying a Flask API

This is a project for the fourth course in the [Udacity Full Stack Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004): Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT.
The built-in Flask server is adequate for local development,
but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).
     
## Final project

Completing the project involves several steps:

### Run the API locally

The following steps describe how to run the Flask API locally with the standard Flask server, so that you can test endpoints before you containerize the app:

#### Install python dependencies

These dependencies are kept in a requirements.txt file. To install them, use pip:

```shell
 pip install -r requirements.txt
```

#### Set up the environment

You do not need to create an env_file to run locally but you do need the following two variables available in your terminal environment.

The following environment variable is required:

 * JWT_SECRET - The secret used to make the JWT, for the purpose of this course the secret can be any string.

 * LOG_LEVEL - The level of logging. This will default to 'INFO', but when debugging an app locally, you may want to set it to 'DEBUG'. To add these to your terminal environment, run the 2 lines below.

 Example:

```shell
 export JWT_SECRET='myjwtsecret'
 export LOG_LEVEL=DEBUG
```

#### Run the app

Run the app using the Flask server, from the top directory, run:

```shell
 python main.py
```

#### Try the API endpoints

Open a new shell and run the following commands, replacing <EMAIL> and <PASSWORD> with any values.

To try the `/auth` endpoint, use the following command:

```shell
export URL=localhost:8080
export TOKEN=`curl -d '{"email":"test@test.com","password":"test"}' -H "Content-Type: application/json" -X POST $URL/auth  | jq -r '.token'`
```

This calls the endpoint `localhost:8080/auth` with the `{"email":"<EMAIL>","password":"<PASSWORD>"} ` as the message body.

The return value is a JWT token based on the secret you supplied. We are assigning that secret to the environment variable 'TOKEN'. To see the JWT token, run:

```shell
echo $TOKEN
```

To try the `/contents` endpoint which decrypts the token and returns its content, run:

```shell
curl --request GET $URL/contents -H "Authorization: Bearer ${TOKEN}" | jq
```

You should see the email that you passed in as one of the values.

### Write a Dockerfile for the API

```shell
FROM python:stretch

COPY . /app
WORKDIR /app

RUN pip install --upgrade pip
RUN pip install -r requirements.txt

ENTRYPOINT ["gunicorn", "-b", ":8080", "main:APP"]
```

### Create a env_file

```shell
JWT_SECRET=myjwtsecret
LOG_LEVEL=DEBUG
```

### Build and test the container locally

```shell
sudo docker build -t jwt-api-test .
sudo docker run -p 80:8080 jwt-api-test
```

### EKS cluster

What is EKS (Amazon Elastic Kubernetes Service)?

* A managed Kubernetes service
* Control layer runs the master system
* Secure networks are set up automatically
* You only setup Nodes, Pods, and Services

In this course, you will be interacting with AWS EKS and other AWS services mostly through the command line interface.

To do this, you will need to install and configure a few command line tools.

#### awscli

[awscli](https://aws.amazon.com/cli/) is the official AWS command line tool.

This tool allows you to interact with a wide variety of AWS services, not just EKS. Although there are aws commands to create or modify EKS services, this is a much more manual approach than using the other options.

* To install in a python virtual environment run `pip install awscli --upgrade`
* Generate keys at the [IAM console](https://console.aws.amazon.com/iam/home#/users).
* Run `aws configure` with the keys from above to setup your profile.
* Check that your profile is set: aws configure list.
* For troubleshooting, have a look at the [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).
* To test you installation, try listing your S3 buckets: `aws s3 ls`. This will show all of the S3 buckets in your account.

#### eksctl

[eksctl](https://eksctl.io) is a simple CLI tool for creating clusters on EKS.

This command line tool allows you to run commands against a kubernetes cluster. This is the best tool for creating or deleting clusters from the command line, since it will take care of all associated resources for you.

```shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

#### kubectl

[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) is commandline tool for interacting with kubernetes clusters.

This tool is used to interact with an existing cluster, but can’t be used to create or delete a cluster.

Download the latest release with the command:

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

```

Make the kubectl binary executable.

```shell
chmod +x ./kubectl
```

Move the binary in to your PATH.

```shell
sudo mv ./kubectl /usr/local/bin/kubectl
```

Test to ensure the version you installed is up-to-date:

```shell
kubectl version --client
```


#### Create an EKS cluster

To clarify, there are two consoles in AWS that can be used to manage EKS services: the EKS console, and the CloudFormation console.

CloudFormation allows you provision all the infrastructure resources that you will need using simple text files, and we will be using CloudFormation going forward in the course.

Create an EKS cluster named `simple-jwt-api`

```shell
eksctl create cluster --name simple-jwt-api
```

Go to the [CloudFormation console](https://us-east-2.console.aws.amazon.com/cloudformation/) to view progress.
If you don’t see any progress, be sure that you are viewing clusters is the same region that they are being created.
For example, if `eksctl` is using region `us-west-2`, you’ll need to set the region to `US West (Oregon)` in the dropdown menu in the upper right of the console.

Once the status is `CREATE_COMPLETE`, check the health of your clusters nodes

```shell
kubectl get nodes
```

If you need to delete the cluster, delete it using eksctl

```shell
eksctl delete cluster simple-jwt-api
```

### Store a secret using AWS Parameter Store

```shell
aws ssm put-parameter --name JWT_SECRET --value "myjwtsecret" --type SecureString
```

### Create a CodePipeline pipeline triggered by GitHub checkins

* Controls the release process through user defined pipelines
* Pipelines are created either through the CodePipeline console or using awscli
* Pipelines watch a source code repository, changes to this repository trigger pipeline action
* Pipelines are made up of stages
* Each stage consists of one or more actions
* There are actions to define the source repository, as well as instructions for testing, building deploying and options for approval
* Pipelines can be managed and viewed in the CodePipeline console

### Create a CodeBuild stage which will build, test, and deploy your code

* Continuous Integration: frequent check-ins to a central repository which trigger automated builds and tests.
* CodeBuild: A fully managed continuous integration system offered by AWS.
* CodeBuild can be added as an action to a CodePipeline stage.

The instructions that a CodeBuild stage will follow are put in a build spec file named `buildspec.yml`.

This file contains all of the commands that the build will run and any related settings.

This file should be placed at the root of your project directory.
Amazon supplies [CodeBuild samples](https://docs.aws.amazon.com/codebuild/latest/userguide/samples.html), you can see examples of build spec files there.

The sample for a simple Docker custom image has the build spec:

You can see that it is divided into the phases ‘install’, ‘pre_build’, and ‘build’. Each phase contains commands, which are the same commands you would use to run Docker locally

```shell
version: 0.2

phases:
  install:
    commands:
      - nohup /usr/local/bin/dockerd --host=unix:///var/run/docker.sock --host=tcp://127.0.0.1:2375 --storage-driver=overlay2&
      - timeout 15 sh -c "until docker info; do echo .; sleep 1; done"
  pre_build:
    commands:
      - docker build -t helloworld .
  build:
    commands:
      - docker images
      - docker run helloworld echo "Hello, World!" 
```