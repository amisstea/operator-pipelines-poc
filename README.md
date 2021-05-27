# operator-pipelines-poc
Proof of Concept for an Operator Certification Pipeline using OpenShift Pipelines.

This proof of concept was put together using Red Hat CodeReady Containers (CRC).
Alternatively, it should be possible to run these pipelines on any OpenShift
cluster of your choosing with minor adjustments.

Additionally, for demonstration purposes, this uses the already established
structure of the Community Operator Pipeline. It should ultimately be triggered
by pull requests made to certified and/or Red Hat Marketplace submission repos.

![Successful pipeline run](img/pipeline-details.png)

## Prerequisites

1. Install [CodeReady Containers](https://code-ready.github.io/crc/#installation_gsg)
2. Install [OpenShift Pipelines](https://docs.openshift.com/container-platform/4.7/cicd/pipelines/installing-pipelines.html)
3. Install the [tkn](https://console-openshift-console.apps-crc.testing/command-line-tools) CLI for your cluster version

## Initial Setup
The following steps assume you have access to a local CRC test cluster and have
logged in via the `oc` CLI (as developer).

1. Create a test project in your cluster

```bash
oc new-project playground
```

2. Clone this repository

```bash
git clone https://github.com/amisstea/operator-pipelines-poc.git
```

3. Apply the OpenShift resources

```bash
oc apply -R -f operator-pipelines-poc/config
```

4. Fork the [community-operators](https://github.com/operator-framework/community-operators)
   repo. Modify one of the operators utilizing the bundle format
   ([example](https://github.com/amisstea/community-operators/commit/88c9c0e4e843e4f5fb34033abb924606017064aa))
   and push your local changes so they are accessible for testing.

## Running the Basic Certification Pipeline

Assuming the working directory is the root of this repo, the base pipeline
(the one a partner may run) can be manually triggered with a Git source
location.

```bash
tkn pipeline start operator-test-pipeline \
  --param git_repo_url=[GIT_REPO] \
  --param git_revision=[BRANCH,COMMIT,TAG] \
  --param bundle_path=[RELATIVE_PATH_WITHIN_GIT_REPO] \
  --workspace name=pipeline,volumeClaimTemplateFile=test/workspace-template.yml
```

Ex:

```bash
tkn pipeline start operator-test-pipeline \
  --param git_repo_url=https://github.com/amisstea/community-operators.git \
  --param git_revision=test-branch \
  --param bundle_path=community-operators/kogito-operator/1.6.0 \
  --workspace name=pipeline,volumeClaimTemplateFile=test/workspace-template.yml
```

That's it! A `PipelineRun` should now be running. View the logs in the
OpenShift console or with the `tkn` CLI.

```bash
tkn pipeline logs -f
```

## Running the Red Hat Certification Pipeline

1. [Optional] Expose your local CRC cluster to the internet, if necessary.
   Tools such as [ngrok](https://dashboard.ngrok.com/get-started/setup) are
   handy for this. This is only required if your cluster cannot be reached
   from beyond your network. For example:

```bash
OCP_ROUTE=$(oc get route operator-cert-ci -o jsonpath='{.spec.host}')
ngrok http --host-header=rewrite $OCP_ROUTE:80
```

2. Setup a GitHub webhook on your fork of `community-operators`. Point this
   hook at your publicly accessible tunnel or OpenShift route. The only
   event type that is supported for now is `Pull requests`.

3. Submit a pull request from your branch to the `master` branch of your
   forked `community-operators` repo. This should trigger the creation of a
   `PipelineRun`.

## Limitations
1. Tekton does not yet support
   [pipelines-in-pipelines](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md).
   This makes composable pipelines difficult. Until this feature is
   readily available, pipeline definitions may be repetitive and not share the
   exact same implementation.
2. Running tasks after a series of conditionally skipped Tekton branches
   (index generation/building) is [not as simple](https://github.com/tektoncd/community/blob/main/teps/0059-skipping-strategies.md)
   as one might expect. [Split-joining](https://github.com/tektoncd/pipeline/issues/3929)
   branches, a common CI/CD workflow, is not available in Tekton's beta state.
   The latter tasks in the pipeline will probably need a solution that's more
   flexible than using the `finally` definition. This may come at the cost of
   added complexity. Pipelines in pipelines may offer a solution to this
   problem.

## Outstanding PoC Work
1. Find a workaround to split-joining with conditional branches
