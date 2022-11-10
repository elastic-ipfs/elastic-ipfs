# Rollback

The production environment must be as protected as possible against problematic releases. However, even when being cautious, and doing all kinds of checks, there is still a chance of things going sideways.

This process have the goal of mitigating the impacts caused by unexpected behaviour, after a newer software version deploy.

## Lambdas

To rollback the code running in lambdas, it's necessary to revert a bad commit/PR that is in the main branch. [Build and Deploy workflows](https://github.com/elastic-ipfs/elastic-ipfs/blob/main/docs/cicd-workflows.md#lambda) will take care of redeploying to stable version. 

## Kubernetes + ArgoCD Rollouts

The [canary deployment strategy](https://argoproj.github.io/argo-rollouts/concepts/#canary) guarantees that only a smaller percentage of users will be affected.

These are the steps to completely remove the unhealthy pods:

- Access [ArgoCD Rollouts dashboard](https://argoproj.github.io/argo-rollouts/dashboard/)
- Select the problematic individual rollout
- Click on `ABORT` button (reference: right top corner)
- Canary pods are terminated
    - At this moment, users are no longer affected by the previous failure
- Rollout is in a `degraded` state, because the real environment no longer matches the desired state
- Commit desired version back to the stable one, this is done in the k8s specs repo, which ArgoCD is syncing from 
- Rollout is back to `healthy` state

Also refer to [FAQ about ArgoCD Rollouts rollbacks](https://argoproj.github.io/argo-rollouts/FAQ/#rollbacks).
