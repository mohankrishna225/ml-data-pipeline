```
docker version
```

![](docker_version.png)

version will show us the client and server engine's version that is currently installed on the machine

![](docker_images.png)

```
docker images
```

This will show images present on the machine

![](docker_pull.png)

```
docker pull <image_name>
```

replace image_name with the name of registry/repo-name:tag, this will pull the image that is present in container registry like dockerhub.com, quay.io or your on-premises registry managed with harbor, we just have to follow the conventio for saving the image name.

![](docker_ps_a.png)

```
docker ps -a
```

this command will list all the containers that are present on the cluster


![](docker_ps.png)

```
docker ps
```

without a ps will just show containers which are currently in running state alone

![](docker_run_a.png)

```
docker run -d --name web-nginx nginx
```

this command will run a container with image: nginx in detached mode with name being web-nginx



![](docker_run.png)

This command will run the container & we can see the logs as well after the container started

![](docker_stop.png)

```
docker stop <container_id>
```

stop command will stop the container and make it to stopped state.

![](docker_rm.png)

```
docker rm <container_id>
```

docker rm will remove the container from the machine

![](docker_rmi.png)

```
docker rmi <image_name>
```

docker rmi will remove the docker image from the machine

![](docker_volume.png)

```
docker volume ls
```

this will list all the volumes present 

```
docker volume create <volume name>
```

this will create a volume for container to store data, we can attach this volume to a container like nginx and save html files 


![](docker_network.png)

this command will create a new docker network where can isolate the containers networking in a separate network namespace

```
docker network create <name>
```

```
docker network ls
```

lists the networks present

![](docker_exec.png)

```
docker exec -it <container_id> bash
```

This is used to login to bash shell of container that is running, helpful in debugging 


![](docker_logs.png)


```
docker logs
```

this helps in viewing the container logs

![](docker_hello_world.png)

```
docker run hello-world
```

this download the image: hello-world from dockerhub and run as a container using docker engine on our machine

![](docker_build.png)

```
docker build -r <registry-name>/repo-name:tag-name
```

this helps us to build a docker image from Dockerfile and -t helps to tag it so that we can identify it and even version the same image with different tags

![](docker_push.png)

```
docker login
```

helps in logging to our repo at dockerhub.com so that we can pull or push to our repo

```
docker push <docker-image>
```

push will authenticate with our login credentials and upload the locally built image to dockerhub.com by default

![](docker_run_fastapi.png)

this is fastapi container running with base image: python3 and runnign our hello-world api on port number 8000



FastAPI Example Dockerfile


```
FROM python:3.8

COPY . /src

COPY ./requirements.txt /src/requirements.txt

WORKDIR src

EXPOSE 8000:8000

RUN pip install -r requirements.txt

#CMD [ "python", "app.py" ]

CMD [ "uvicorn", "app:app", "--host", "0.0.0.0", "--reload" ]
```


```
from fastapi import FastAPI

app = FastAPI()

@app.get("/")

def read_root():

return { "Hello World": "Welcome to MLOps Training" }

```

Docker Image: mohankrishna225/fastapi:helloworld

Repo: https://github.com/mohankrishna225/fastapi-docker


Create a New Workflow 

![image](https://user-images.githubusercontent.com/45117258/196667599-76edace3-ade7-457f-b0a6-cf53115ebd08.png)


we can specify the workflow with a job to build docker image and push it to dockerhub as stages inside the job as follows, and not to forget about when to start this build action is specified with help of ON Section, we have specified it to be on push command to main branch with tags v 

Also we need to configure the secrets used this workflow like DOCKER_TOKEN in your repository so that it's secure 

![image](https://user-images.githubusercontent.com/45117258/196668184-3392aa65-c01f-47b2-a198-b94ae2b8314b.png)


```
name: Build Docker Image and push to dockerhub

on:
  push:
    branches: [ "main" ]
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ "main" ]

env:
  REGISTRY: docker.io
  # github.repository as <account>/<repo>
  IMAGE_NAME: ${{ github.repository }}


jobs:
  build:

    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install cosign
        if: github.event_name != 'pull_request'
        uses: sigstore/cosign-installer@f3c664df7af409cb4873aa5068053ba9d61a57b6 #v2.6.0
        with:
          cosign-release: 'v1.11.0'


      - name: Setup Docker buildx
        uses: docker/setup-buildx-action@79abd3f86f79a9d68a23c75a09a9a85889262adf


      - name: Log into registry ${{ env.REGISTRY }}
        if: github.event_name != 'pull_request'
        uses: docker/login-action@28218f9b04b4f3f62068d7b6ce6ca5b26e35336c
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}


      - name: Build and push Docker image
        id: build-and-push
        uses: docker/build-push-action@ac9327eae2b366085ac7f6a2d02df8aa8ead720a
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max


      - name: Sign the published Docker image
        if: ${{ github.event_name != 'pull_request' }}
        env:
          COSIGN_EXPERIMENTAL: "true"

        run: echo "${{ steps.meta.outputs.tags }}" | xargs -I {} cosign sign {}@${{ steps.build-and-push.outputs.digest }}
```
