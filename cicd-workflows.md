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


 ## Deploy Workflow

 This is triggered when a build workflow has finished running successfully for the long-lived branches. Each of these is associated with the environments that should be updated.

- dev: dev (Automatic)
- main: Staging (Automatic) and Prod (Requires manual approval)

The steps for it will vary depending if component should be deployed to a lambda or Kubernetes.

 ### Lambda

This diagram shows the complete process of introducing a new feature for a lambda into all three environments.

![Lambda Workflow](assets/images/workflows-lambdas.png)

This is the main step
 ```
Run Deploy
```

That step means that after the AWS credentials have been properly configured, the lambda update-function-code is executed with the docker image tag that has been provided by `build workflow`.

#### Security 

[AWS is configured to trust GitHub's OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services). That means repos work with tokens instead of storing long-lived AWS credentials in secrets.


 ### Kubernetes

 This diagram shows the complete process of introducing a new feature for an application running in Kubernetes into all three environments.


![Lambda Workflow](assets/images/workflows-kubernetes.png)

It will have at least these major steps:
 ```
Clone Helm Deployment Repo => Update Image Tag => Commit and push 
```

The only thing that actually happens at this GH workflow is to update the image tag in a helm values spec, using the value which came from build workflow.

Applications which run on Kubernetes have their specs managed by helm charts. Those are stored in another repo with `**-deployment` sufix. Example: [`ipfs-elastic-provider-bitswap-peer`](https://github.com/ipfs-elastic-provider/ipfs-elastic-provider-bitswap-peer) has a correspondent [`ipfs-elastic-provider-bitswap-peer-deployment`](https://github.com/ipfs-elastic-provider/ipfs-elastic-provider-bitswap-peer-deployment) repo.

This follows a [GitOps Pull Model approach](https://dzone.com/articles/why-is-a-pull-vs-a-push-pipeline-important), which means that GH doesn't push to K8S.

ArgoCD is running inside the cluster monitoring possible changes on its correspondent environment spec. For example: ArgoCD in K8S dev is always syncing with `values-dev.yaml` file.


When an image tag is updated in the `values-<env>.yaml` file, ArgoCD automatically knows  how to change the `deployment` spec, so that new pods can be created.


#### Security

There is no need of configuring any kind of access from GitHub, it stores zero long or short lived credentials.


## Shared Workflows

[This repo](https://github.com/ipfs-elastic-provider/shared-workflows) stores generic workflows that are reused by several components.

## Important Notes

- Keep `main` and `dev` branches as sync as possible. 
- Keep `staging` and `prod` environments as sync as possible. Remember you **can't** add new code that is just for `prod` without releasing everything that's deployed in `staging`.
- Don't forget to merge/rebase from the target branch when doing PR.
- As stated by the [Gitflow Workflow strategy](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow), we can hotfix by creating PRs directly from `main`.
- Avoid been too bureaucratic about this process. If you're building a new feature that doesn't make sense going to `dev` first (for whatever reason), open the PR directly from `main` and **be happy**
