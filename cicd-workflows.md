# CI/CD Workflows

All Elastic Provider component repositories follow the [Gitflow Workflow strategy](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow). They can have any number of short-lived feature branches and two long-lived branches, which are: `dev` and `main`.

## Build Workflow

When new changes are introduced to the long-lived branches, they trigger a build workflow.

It will have at least these major steps:

```
Run Test => Run Lint => Build Artifact (Docker Image) => Publish Artifact
```

### PR

 All pull requests from a feature branch (or hotfix) to the long-lived branches also trigger a workflow. It has the same steps described above, except for the last one: it doesn't publish any artifacts. The focus is validation and quality. This guarantees a **fail fast** strategy, making possible to identify problems as soon as possible.

 ```
Run Test => Run Lint => Build Artifact (Docker Image)
```


 # Deploy Workflow

 This is triggered when a build workflow has finished running successfully for the long-lived branches. Each of these is associated with the environments that should be updated.

- dev: dev (Automatic)
- main: Staging (Automatic) and Prod (Requires manual approval)

The steps for it will vary depending if component should be deployed to a lambda or Kubernetes.

 ## Lambda

![Lambda Workflow](assets/images/workflows-lambdas.png)

 ==> TODO: Talk about OIDC security


 ## Kubernetes

![Lambda Workflow](assets/images/workflows-kubernetes.png)

 ==> TODO: No credentials required


 # Shared Workflows

 ==> TODO: Just link with that repo