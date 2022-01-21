# CI/CD

This repo is about microservices and testing them in addition to setting up a build and deploy pipeline that gets working code into a live environment.

## CI/CD pipeline with github actions

Agenda:
  - Create Github Repo
  - Simple REST Endpoint and Test
  - Build Pipeline Initialization
  - Docker Build and Registry
  - Helm Initialization and Chart Publishing
  - Setup Cloud Hosting
  - Automate Kubernetes Deployment

## Feature Branches

Executed on every push to [feature](.github/workflows/features.yaml) branches:
  - Build and run tests
  - Extract git repo metadata
  - Build and publish docker image (`:docker_build@sha256...`)

## Main Branch

Kicked off on every push to [main](.github/workflows/main.yaml) branch only:
  - Build and run tests
  - Tag git repo with new version (`major.minor.patch`)
  - Extract git repo metadata
  - Build and publish docker image (`:version@sha256...`)
  - Publish helm chart
  - Deploy helm chart to Kubernetes cluster

Service got deployed to an Okteto namespace in this case.

## Versioning

Push up a new major version:

```sh
git tag -a v2.0.0 -m "pushing major version"
git push origin v2.0.0
```
