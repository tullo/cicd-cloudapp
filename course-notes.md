# Notes

This repo is about writing spring boot based microservices and testing them in addition to setting up a build and deploy pipeline that will get your working code into a live environment.

I’ll show how to build the CI/CD pipeline with github actions.

Agenda:

  - Create Github Repo
  - Simple REST Endpoint and Test
  - Build Pipeline Initialization
  - Docker Build and Registry
  - Helm Initialization and Chart Publishing
  - Setup Cloud Hosting
  - Automate Kubernetes Deployment

---

## Create Github Repo

```sh
git init
git remote add origin git@github.com:<org>/<repo>.git
git fetch
git checkout origin/main -ft  # (--force --track)

# Skaffold spring-boot project
git update-index --chmod=+x ./mvnw
git add .
git commit -m 'init project'
git push
```

---

## Simple REST Endpoint and Test

Create a Basic Microservice Endpoint.

```sh
touch src/main/java/../../../controller/HelloWorldController.java

touch src/main/java/../../../controller/dto/HelloWorldResponse.java

# Automated testing
touch src/test/java/../../../controller/HelloWorldControllerTest.java
./mvnw -B package

# Create feature branch and push
git checkout -b microservice_dev
git add .
git commit -m "add rest endpoint and test"
git push --set-upstream origin microservice_dev
```

---

## Build Pipeline Initialization

### Initialize Github Actions

```sh
touch .github/workflows/features.yaml
# - Execute this on every push except if the push is to master.
#   This will build on every commit/push.
#   This will tell us if we broke the build.
# - Checkout our git repo
# - Setup a build environment (ubuntu with java 11).
#   Cache the maven dependencies.
# - Run the maven build with test.

git add .
git commit -m "add build pipeline"
git push --set-upstream origin microservice_dev
```

Github Actions should go green by now.

### Create the Main Build and Deploy

```yaml
# touch .github/workflows/main.yaml
# - Same steps as in features build pipeline.
# - Update build trigger.
on:
  push:
    branches:
      - main
```

Commit and push this change. Should only trigger a build on the feature branch (because it’s a commit to a non main branch).

```sh
git add .
git commit -m "main build"
```

Merge microserivce_dev into main.

```sh
git checkout main
git merge --squash microservice_dev

git commit -m "add microservice endpoint and build pipeline"
git push
```

Github Actions should show a single build being kicked off from the main branch.

---

## Docker Build and Registry

### Create a Docker Image Registry

[Artifactory as Your Kubernetes Registry](https://www.jfrog.com/confluence/display/JFROG/JFrog+Artifactory)
- Create an account for free
- Setup a docker repository [packages/docker://cloudapp-java](https://<namepsace>.jfrog.io/ui/packages/docker:%2F%2Fcloudapp-java)

Github settings
- Add secret JFROG_USER
- Add secret JFROG_ACCESS_TOKEN

### Create a Dockerfile

```Dockerfile
FROM openjdk:11-slim-buster as build

COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .
COPY src src

# build and cache maven dependencies
RUN --mount=type=cache,target=/root/.m2,rw ./mvnw -B package -DskipTests

FROM openjdk:11-jre-slim-buster
COPY --from=build target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Add Docker Build to our Pipeline.

When we build on a feature branch we’re tagging the image with a build tag by default.

```yaml
jobs:
  build:
    steps:

      # ^^^ skipped previous steps ^^^

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to jfrog.io
        uses: docker/login-action@v1
        with:
          registry: <namespace>.jfrog.io/default-docker-virtual
          username: ${{ secrets.JFROG_USER }}
          password: ${{ secrets.JFROG_ACCESS_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v3
        with:
          images: <namespace>.jfrog.io/default-docker-virtual/cloudapp-java
          tags: |
            type=ref,event=tag

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

Push this up on a feature branch to see if it builds:

```sh
git checkout -b docker_build
git add .
git push --set-upstream origin docker_build
```

It's going to take a long time the first time because EVERYTHING has to be pulled down, cached and built the first time.

Subsequent builds shouldn’t take as long.

### Create the Main Build Pipeline

When we build off of main branch we’re going to want a semantic version.

```yaml
jobs:
  build:
    steps:

      # ^^^ skipped previous steps ^^^

      - name: Bump version
        run: |
          git config --global user.email "github+actions@gmail.com"
          git config --global user.name "Actions"
          git fetch --tags
          wget -O - https://raw.githubusercontent.com/treeder/bump/master/gitbump.sh | bash
          echo "VERSION=$(git tag --sort=-v:refname --list "v[0-9]*" | head -n 1 | cut -c 2-)" >> $GITHUB_ENV

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: |
            <namespace>.jfrog.io/default-docker-virtual/cloudapp-java:${{ env.VERSION }}
```

Lets try it out.

```sh
git switch main
git add .
git commit -m "update main docker pipeline"
git push

git merge --squash docker_build
git commit -m "add docker build pipeline"
git push
```

### Versioning

Switch back over to your github actions and watch the main pipeline build.

You should see something like this in your build logs:

```sh
16 exporting to image
16 pushing layers 3.1s done
16 pushing manifest for <namepsace>.jfrog.io/default-docker-virtual/cloudapp-java:1.0.7@sha256:e68a6459f092d2d37bb1385f46c276807a79ae1bb4f80a217b1edf5b06ab4b36
16 pushing manifest for <namepsace>.jfrog.io/default-docker-virtual/cloudapp-java:1.0.7@sha256:e68a6459f092d2d37bb1385f46c276807a79ae1bb4f80a217b1edf5b06ab4b36 1.8s done
```

The image got tagged with `1.0.7`.

And you can push up a new major version
```sh
git tag -a v2.0.0 -m "pushing major version"
git push origin v2.0.0
```

Make a code change and you’ll see that the version change gets picked up and incorporated.

---

## Helm Initialization and Chart Publishing


`helm install --dry-run --generate-name . > dep.yaml`

helm install --generate-name .

---

## Setup Cloud Hosting

### Setup Okteto

- Register on okteto by creating an account via your github account.
- Download okteto cli
- Login with okteto cli
- Merge okteto with existing k8s config

Make sure current context points to okteto

```sh
kubectl config current-context
# cloud_okteto_com
```

Setup regcred secret so that okteto can access our images in frog.io

```sh
kubectl create secret docker-registry regcred \
  --docker-server=${your_registry} \
  --docker-username=${your_username} \
  --docker-password=${your_password} \
  --docker-email=${your_email}
```

### Deploy Manually with Helm

Manually deploy into okteto via helm on the command line.

`helm repo add tullo https://tullo.github.io/helm-charts/`

Update the repo:

`helm repo update tullo`

Double check your repo:

`helm search repo tullo`

Deploy it:

`helm install --generate-name tullo/cloud-application-java`

You should see your service deployed in okteto.

Grab the url from the endpoints section and copy that to confirm you can access it:

```sh
curl -s https://cloud-application-java-<namespace>.cloud.okteto.net/helloworld | jq
# {
#   "message": "Hello World!"
# }
```

Anytime that we make an update into the main branch now, we should be getting an updated chart pushed to our repository.

We can deploy the updated chart into okteto with the following commands:

```sh
helm repo update tullo
helm list

# NAME            	NAMESPACE	REVISION	CHART                       	APP VERSION
# chart-1642000328	tullo    	1       	cloud-application-java-0.1.0	1.0.7

helm upgrade chart-1642000328 tullo/cloud-application-java

## [pod-event] Successfully pulled image "tullo.jfrog.io/default-docker-local/cloudapp-java:1.0.9"

helm list

# NAME            	NAMESPACE	REVISION	CHART                       	APP VERSION
# chart-1642000328	tullo    	1       	cloud-application-java-0.1.0	1.0.9
```

---

## Automate Kubernetes Deployment

Update main pipeline to deploy on successful build.

```yaml
      - name: Deploy
        uses: WyriHaximus/github-action-helm3@v2
        with:
          exec: |
            helm repo add tullo https://tullo.github.io/helm-charts/
            helm repo update
            helm upgrade chart-1642000328 tullo/cloud-application-java --install --atomic
          kubeconfig: '${{ secrets.KUBECONFIG }}'

```

This will ppgrade the cloud instance if it exists, otherwise install (`--install`) or rollback if it fails to upgrade (`--atomic`).

### Add the Okteto Kube Config

Create a new GH secret called KUBECONFIG paste the entire contents of the okteto context file in as the value.

- `okteto context update-kubeconfig`
- `cat /home/anda/.kube/config`

### Redeploy and Verify

Push your changes up to main

```sh
git add .
git commit -m "automate helm chart deployment"
git push
```

Now navigate back to your github actions tab. Confirm that the build kicked off.

Under `Publish Helm Chart` you should see a new version got created:

`Successfully packaged chart and saved it to: /tmp/tmp.kfHbLe/cloud-application-java-1.0.10.tgz`

Under `Deploy` you should see that it deployed:

```sh
Release "chart-1642000328" has been upgraded. Happy Helming!
NAME: chart-1642000328
LAST DEPLOYED: Thu Jan 13 08:38:47 2022
NAMESPACE: tullo
STATUS: deployed
REVISION: 2
NOTES:
...
Cleaning up: 
  - exec ✅ 
  - kubeconfig ✅ 
```

Navigate to okteto and expand the deployment, check the yaml to see if the new version was deployed

https://cloud.okteto.com/#/spaces/

Double check endpoint access:

```sh
curl -s https://cloud-application-java-<namespace>.cloud.okteto.net/helloworld | jq
# {
#   "message": "Hello World!"
# }
```

There you go!
- Now any changes made into a feature branch will build and test.
- When you have confidence it can release, you can merge to main and it will get deployed into your cloud hosted environment.

This makes updating your application much easier. However, it can be made much easier with tools like [skaffold](https://skaffold.dev/) that hook into your IDE and build and push changes automatically after code changes.

---

Credit:
- https://bullyrooks.com/index.php/2022/01/04/cloud-kube-setup-cloud-hosting/
- https://github.com/bullyrooks/cloud_application
