# Continuous Deployment for Testnets following GitOps

Continuous Deployment (CD) are a set of practices aimed at increasing the security, efficiency and overall reliability of the deployment process by introducing GitOps automation. These can be adopted in the deployment process of Mina testnets, enabling enhanced accountability, reliable deployments, and configuration drift prevention.

## Summary

Declarative models are particularly useful when managing Infrastructure as Code (IaC) and Kubernetes deployments, mainly because these are used to describe a ***desired state***.

By leveraging Git as the single source of truth holding the desired state of infrastructures or deployments, a great deal of automation may be built around events to such state. This is essentially what the GitOps practices proposes:
- Git as single source of truth
- Tooling (i.e., **GitOps Operator**) able to monitor/react to changes to the source of truth and apply them on a destination infrastructure (e.g., GCP, GKE, Kubernetes, etc.)
- Keep the current state of the deployment in line with the desired state (i.e., configuration drift prevention).

In a Software Development Life Cycle (SDLC) a GitOps Continuous Integration (CI) workflow can be generalized in the following pipeline:

```text
┌──────────┐      ┌────────┐      ┌──────────────────────────┐
│ 1. Build │ ---> │ 2. Test│ ---> │ 3. Container image Push  │──┐
└──────────┘      └────────┘      └──────────────────────────┘  │
 ┌--------------------------------------------------------------┘
 │   ┌──────────────────────────┐      ┌─────────────────────┐
 └-> │ 4. Git clone config repo │ ---> │ 5. Update manifests │──┐
     └──────────────────────────┘      └─────────────────────┘  │
 ┌--------------------------------------------------------------┘
 │     ┌────────────────────────┐
 └---> │ 6. Git commit and push │
       └────────────────────────┘
```

Where: 
1. A container image is **built**. This generally includes unit testing and code smell analysis.
2. Functional tests continue.
3. Once testing phase is cleared, the image is pushed to a repository with an specific version tag.
4. The deployment configurations repository is fetched as to
5. update deployment manifests with new version tag.
6. The change is pushed to the deployment configuration repository.

Steps `[4-5]` are key to a GitOps Continuous Deployment workflow. In the example, the referenced `config repo` represents the desired state of a particular deployment. Therefore, a CD workflow will be triggered by such change:

```text
┌──────────────────────────┐      ┌───────────────────────┐    
│ 7. Git clone config repo │ ---> │ 8. Discover manifests │ ─┐
└──────────────────────────┘      └───────────────────────┘  │
┌------------------------------------------------------------┘
│   ┌───────────────────┐
└-> │ 9. kubectl apply  │
    └───────────────────┘
```
This latter workflow reflects the actions taken by the GitOps Operator, these are:

7. Cloning `config repo` upon changes.
8. Discover changes to monitored manifests.
9. Apply new configuration so current state matches desired state.


The overall architecture proposed by this model is shown below:
![Generic Architecture](img/0011/gitOps-arch.svg "Generic Architecture")

## Motivation

The above architecture proposes the following features:
- GitOps Operator deals with the deployment, configuration drift protection and rollback tasks.
- Enables Teams to create, review and/or make changes to deployments without need to understand the underlying procedures.
  - Changes are reviewed and approved by authorized parties via Git-native mechanisms, such as pull-requests.
  - Baked into the process is increased accountability or transparency. Anyone may review and understand reasoning of changes by reading commit messages and participating on pull-requests discussions.
  - Authorized parties then trigger deployment actions by merging commits to corresponding branches.
- Infrastructure access is secured as only GitOps Operator may perform changes.

All in all, the proposed architecture/workflow ensures that Platform Engineering focuses on developing and maintaining infrastructure tools. Consequently, such tools empower development teams to change the system's desired state via Git, supported by a review process in which cross-collaboration is possible and transparency is built-in.

## Objective
Implement a GitOps-based Continuous Deployment PoC for testnets.

### Detailed objectives

#### 1. Design and develop a Generic GitOps CD PoC
This will involve the design and deployment of a GitOps CD workflow. Relevantly, it should take into consideration the management and operation of Secrets and how these would be injected securely into Helm Charts before deployment. Further, it should include a GitOps Operator able to achieve the features proposed by the architecture.

#### 2. Adapt workflow to include testnet Charts' external requirements
Currently, deployments retrieve data from secured vaults (e.g., in GCP) in order to pre-configure the deployment's namespace with all required Secrets. This task will detail such requirements and adapt the workflow design as to satisfy them.

#### 3. Deploy a Testnet following a GitOps CD workflow
Attempt a deployment of a sample dummy testnet leveraging the designed workflow.

## 1. Generic GitOps CD PoC
Based on the architecture, the following elements are identified:

- ### GitOps Operator
  Maintains a series of reconciliation loops enforcing the desired state (`source` of truth i.e., deployment manifest at Git) on the `destination` cluster. The following shows an instance of the architecture where a Pull-Request (PR) scaling the replicas for a deployment is merged into `config repo`.
  ![Generic Scaling](img/0011/generic-scaling.svg "Generic Scaling")
  
  Also, if `destination` drifts from `source` (e.g., due to manual edits on the current state), it applies/reverts changes to achieve successful reconciliation. This is shown in the Figure below, where the `source` and `destination` Reconciliation Loops represent the desired and current states, respectively.

  ![Generic Drift](img/0011/generic-drift.svg "Generic Drift")

  One widely used GitOps Operator is [ArgoCD](https://argo-cd.readthedocs.io/en/stable/).

- ### Git 
  this refers to the overall git repository structure supporting CD. Best practices encourage to separate source code from deployment configurations, as to prevent development commits from unintentionally trigger/modify deployment configurations. Relevantly, this practice allows for the introduction of **Deployment Templates**, that is, desired state produced by CI pipelines.

- ### Secrets Operator 
  To accurately reflect the desired state of a deployment following GitOps a mechanism for retrieving and storing Secrets (alongside the desired state) is required. A Secrets Operator may enable this by different means. For example: 

  [Bitnami's Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) deploys an in-cluster encryption infrastructure. Encrypted `sealed secrets` when applied to the cluster signal the Secrets Operator Controller to produce a linked Kubernetes Secret with the values decrypted. Sealed Secrets can be stored in git as they can only be decrypted by the Sealed Secrets Controller and used on an specific namespace. 
  
  An alternative is the [External Secrets Operator](https://external-secrets.io/latest/) (ESO), whose Controller is able to synchronize data from external APIs (e.g., Google Secrets Manager, AWS Secrets Manager, etc.) and inject it into Kubernetes Secrets. Such `ExternalSecrets` may also be stored in git as they do no hold sensitive data, only reference to a remote `SecretStore` and to the key of the value to be injected into the resulting Secret.
  
  A Secrets Operator is required for producing the desired state of a deployment:
  - Secured Secrets (via Sealed Secrets or ESO) equivalents for each Kubernetes Secrets per deployment must be produced and included in the corresponding deployment repository.
  - For the Sealed Secrets scenario: access to the Secrets Operator is required to produce the `sealed secrets`. [Offline production of Sealed Secrets is possible](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md), but operational cost may increases with the management of certificates.
  - ESO decouples the population of the `SecretStore`, in fact, allows to have more than one secret store.

### Generic PoC Definition
The PoC will implement the proposed architecture instance. Relevantly, it will:
- Leverage ArgoCD as a GitOps Operator.
- Use External Secrets Operator (ESO) as Secrets Operator, reusing the existing external secret store (i.e., GCP Secrets Manager).
- Deployment configuration (i.e., Helm `values.yaml` or deployment Charts) are to be stored on an independent repository, where different deployments are clearly identified.

#### Comments
This scenario has already been tested. The following YAML represents the ArgoCD Application (i.e., the definition of Reconciliation Loops):
```yaml
# Some rows ommitted for brevity
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ci-dashboard-app
...
spec:
  info:
  - name: "CI Dashboard PoC:"
    value: "Periodically retrieving metrics from Buildkite and Github. Populating a local Postgres DB which serves as datasource for Grafana."
  source:
    repoURL: "git@github.com:MinaProtocol/mina.git"
    targetRevision: luis-ci-dashboard
    path: helm/ci-dashboard
    helm:
      releaseName: ci-dashboard
      values: |
        postgresql:
          primary:
            persistence:
              enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: ci-dashboard
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5 # number of failed sync attempt retries; unlimited number of attempts if less than 0
      backoff:
        duration: 15s # the amount to back off. Default unit is seconds, but could also be a duration (e.g. "2m", "1h")
        factor: 2 # a factor to multiply the base duration after each failed retry
        maxDuration: 10m # the maximum amount of time allowed for the backoff strategy
    syncOptions:
    - CreateNamespace=true
```
Secrets involved in the PoC include:
- Credentials to checkout git sources
- Credentials to onboard destination clusters.

Both were secured using Sealed Secrets. An implementation with ESO is pending.

## 2. Implementing GitOps CD for Testnet
