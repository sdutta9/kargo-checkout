# Kargo Quickstart Template

These are the supporting files for [the Kargo Quickstart tutorial](https://docs.akuity.io/tutorials/kargo-quickstart/) in the Akuity docs.

This tutorial will walk you through a working example using Kargo with the Akuity Platform, to manage the promotion of an image from stage to stage in a declarative way.

Ultimately, you will have a Kubernetes cluster, with Applications deployed using an Argo CD control plane; and handle promotion with Kargo.

# Important commands to run in Github codespace when you resume work

```bash

export KARGO_QUICKSTART_PAT=<your github pat>

echo $KARGO_QUICKSTART_PAT
```
```bash
docker login --username ${GITHUB_USER} --password ${KARGO_QUICKSTART_PAT} ghcr.io
```
```bash
kargo login <instance_url> --admin --password <password>

```

# Updates needed in Kargo quickstart Docs

## Section 3.2.3 Kargo Stages

- Doc uses `promotionMechanisms` where as the github template repo has more latest spec that makes use of `promotionTemplate`

## Section 3.3 Kargo Promotion

- Stages.yaml file has below argoCD step that fails as at this point we donot have a ArgoCD project to which Kargo can update

    ```yaml
    - uses: argocd-update
        config:
        apps:
        - name: guestbook-simple-dev
            sources:
            - repoURL: ${{ vars.gitRepo }}
            desiredRevision: ${{ outputs.commit.commit }}
    ```
    **Workaround:** Comment above section in dev, staging and prod stages for now.

## Section 4.3.2 Update Kargo Stages

Do not run `scripts/kargo-argocd-manifestupdate.sh` script as it references the old stages files using `promotionMechanisms`. Instead the sections that was commented out in the above section needs to be uncommented. Created two seperate manifest files (`stage_non-prod.yaml` and `stage_prod.yaml`). Make sure you reapply those two files to update your dev, staging and prod stages with `argocd-update` step.



# Bonus Implementation: PR-Gated Production Promotions

As part of the optional enhancements, this repository implements a distinct, human-in-the-loop promotion strategy for the production environment utilizing Kargo's native Pull Request automation.

## Architectural Design

While lower environments (dev and staging) are configured for continuous, automated deployments via direct branch pushes, the prod stage introduces a required control gate. Instead of pushing changes directly to the production tracking branch, Kargo is configured to dynamically generate a feature branch, open a Pull Request against the env/prod branch, and pause the promotion pipeline until the PR is successfully merged by an authorized user.

## How the Pipeline Operates

1. **Dynamic Branching:** When a promotion to prod is triggered, Kargo renders the manifests and uses the generateTargetBranch: true configuration to push these changes to a unique, temporary Git branch.

1. **Automated PR Creation:** Kargo interfaces with the GitHub API to open a Pull Request targeting the env/prod branch, detailing the automated changes.

1. **Pipeline Suspension:** The git-wait-for-pr step pauses the Kargo pipeline, effectively blocking Argo CD from seeing any changes until a human reviews and merges the PR.

1. **Argo CD Sync:** Once the PR is merged into env/prod, the pipeline resumes, and Argo CD synchronizes the approved, raw YAML directly to the production cluster.


## Design Decisions

- **Security & Compliance:** Enforcing a PR approval process for production changes satisfies standard enterprise compliance frameworks (such as SOC2) by guaranteeing separation of duties and mandatory peer review before production state mutation.

- **Developer Experience:** By automating the creation of the branch and the PR itself, the pipeline removes the manual toil usually associated with GitOps PR workflows, leaving only the high-value task (the actual review and merge) to the engineering team.


## Tradeoffs

**Increased Lead Time:** Introducing a manual PR review slows down the absolute speed of deployment to production compared to the fully automated lower environments.