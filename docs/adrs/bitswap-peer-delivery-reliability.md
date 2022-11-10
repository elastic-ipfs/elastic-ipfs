# Bitswap Peer Delivery Reliability

Contents:

- [Bitswap Peer Delivery Reliability](#bitswap-peer-delivery-reliability)
  - [Summary](#summary)
    - [Issue](#issue)
    - [Options for Rollouts](#options-for-rollouts)
    - [Options for GitOps](#options-for-gitops)
    - [Decision](#decision)
    - [Status](#status)
  - [Details](#details)
    - [Constraints](#constraints)
    - [Argument](#argument)
    - [Implications](#implications)
  - [Notes](#notes)


## Summary

Our [current delivery process for k8s deployments](https://github.com/elastic-ipfs/elastic-ipfs/blob/main/cicd-workflows.md#kubernetes) relies on:

- [Default rolling update strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- Running app is synced with main branch code
- Reverse commit for rollback

The production environment must be as protected as possible against problematic releases. We can strength our existing safeguards by:

- Improving how we rollout/rollback
  - Consider adopting [progressive delivery](https://argoproj.github.io/argo-rollouts/concepts/#progressive-delivery)
- Improving GitOps automation processes

### Issue

Introducing changes to Bitswap Peer can lead to production downtime. Previously successfully running pods get replaced by broken ones. 

Example scenarios:

- Unexpected extreme increase of compute resources consumption ([OOMKilled](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/))
- Missing NPM package
- A feature change introduces unexpected business behavior/logic

We already have `dev` and `staging` environments as our most important lines of defense. This means that **all these problems should be captured before promoting to production**. Unfortunately, things can still go wrong, a problem bypasses these first two big walls and invades town.


### Options for Rollouts
- ArgoCD Rollouts
- Improve current k8s rolling update strategy configuration
  - Tweak strategy fields. 
    - Increase `minReadySeconds`
    - Reduce `maxUnavailable`
    - Increase `maxSurge`
  - Might still be problematic in memory leak scenarios
  - Doesn't allow progressive delivery
  - Doesn't have automatic rollback
- Istio Service Mesh
  - Allows progressive delivery but also comes with lots of other capabilities which we don't need in our single app scenario
  - Overkill
- Weighted round robin in DNS
  - Could achieve progressive delivery by using DNS record weights to different k8s services (current and new version)
  - Platform agnostic
  - Always require manual promotion
  - We can't control client DNS cache
  - Supported by AWS Route53 but not by CloudFlare. It's possible to have similar behavior by using a CloudFlare weighted load balancer
- Migrate Bitswap Peer from EKS to ECS
  - Managed services
  - Requires using AWS CodeDeploy service for progressive delivery
  - Requires further research

### Options for GitOps

Those are not mutually exclusive.

- Deployment repo as source of truth rather than application repo
- Adopt Tag Releases
  - Used by many open source projects. Commonly associated with semantic versioning
  - A release document is a great way of knowing what has changed between releases
  - Requires creating the new release manually to trigger workflow
  - Hotfix complexity
  - If adopted together with the first option, this tag release process would be used in the application repo, ending at "publish new image version to registry" step. No environments updated

### Decision

For rollouts, we've decided to use [ArgoCD rollouts](https://argoproj.github.io/argo-rollouts/) because: 

- It allows progressive delivery. We can do canary updates
- It allows manual or automated rollback and promotions
  - Automation is based on metrics which we choose
  - Prometheus integration
  - Speed control
- Traffic management can be done without any extra component integration required
  - There are optional integrations with ALB or NGINX operators.
- Compatibility of [`rollout` resource](https://argoproj.github.io/argo-rollouts/features/specification/) with `deployment`. Easy to migrate and start using the extended fields
- We are already using ArgoCD to deploy

For GitOps, we've decided to use deployment/configuration repo as source of truth for the environment rather than application one because:
- It's a decoupled solution. It improves responsibility segregation
- It enables deploying this application to different platforms. It also expands the possibilities of how to do so (helm, kustomize, argocd, GH actions...)

### Status

- Reviewed and approved by team
- Implementation phase

## Details

We can implement this process through a phased approach:

### Phase1: Reliability and Decoupling
  1. Install ArgoCD Rollouts Operator
  2. Adapt Bitswap Peer `deployment` resource to `rollout` using canary strategy. Configure steps that require manual promotion
  3. Remove existing automatic push to deployment/configuration repository when a new image is available. Will require manual updates to helm values file 

Expected experience:

- Application Repo
  - Changes merged to main branch
  - Workflow does all quality checks and pushes image to remote registry 
- Deployment/configuration Repo
  - Create PR with the desired image version
  - Review, approve and merge of PR
- ArgoCD syncs with desired state. Rollouts makes sure to replace only the percentage of pods configured as first canary step.
- Monitor behavior of canary pods in production
- If everything is OK, promote!
- If not Ok :bomb:, abort (auto rollback) :runner: .
  - Also revert commit in deployment/configuration repo

### Phase 2: Automation

- Phase2: Automation
  1. Subscribe deployment repo to GH Webhook which notifies newer image version. Automatically update helm spec values
  2. Adopt metrics analysis for achieving automated promotion during canary deploy
  3. Traffic management based on ALB operators (if using it)


### Constraints

- Continue to use k8s as our container orchestrator
- k8s version should be `v1.23` or higher. Found version mismatch of authorization APIs when running rollouts in `v1.21`

### Argument

ArgoCD rollouts provides the delivery reliability we expect through canary deployments. It also provides good experience of managing them through CLI or dashboard.

### Implications

- This process slow down release speed
- It requires the installation and management of a new operator in k8s
- It's easier to bypass staging environment, so it requires maturity of the team and of the process
- Increasing complexity (and possible generating confusion) of how to update an environment

## Notes

None