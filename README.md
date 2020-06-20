# Deploying a Flask API

This is the project starter repo for the fourth course in the [Udacity Full Stack Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004): Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Initial setup
1. Fork this project to your Github account.
2. Locally clone your forked version to begin working on the project.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).
     
## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson [here](https://classroom.udacity.com/nanodegrees/nd004/parts/1d842ebf-5b10-4749-9e5e-ef28fe98f173/modules/ac13842f-c841-4c1a-b284-b47899f4613d/lessons/becb2dac-c108-4143-8f6c-11b30413e28d/concepts/092cdb35-28f7-4145-b6e6-6278b8dd7527).

## Steps to run the API locally using the Flask server (no containerization)

The following steps describe how to run the Flask API locally with the standard Flask server, so that you can test endpoints before you containerize the app:

### Install python dependencies. 

These dependencies are kept in a requirements.txt file. To install them, use pip:

```shell
 pip install -r requirements.txt
```

### Set up the environment. 

You do not need to create an env_file to run locally but you do need the following two variables available in your terminal environment.

The following environment variable is required:

 * JWT_SECRET - The secret used to make the JWT, for the purpose of this course the secret can be any string.

 * LOG_LEVEL - The level of logging. This will default to 'INFO', but when debugging an app locally, you may want to set it to 'DEBUG'. To add these to your terminal environment, run the 2 lines below.

 Example:

```shell
 export JWT_SECRET='myjwtsecret'
 export LOG_LEVEL=DEBUG
```


### Run the app

Run the app using the Flask server, from the top directory, run:

```shell
 python main.py
```

### Try the API endpoints

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