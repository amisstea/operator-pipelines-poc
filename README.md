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

## Setup
The following steps assume you have access to a local CRC test cluster and have
logged in via the `oc` CLI (as developer).

1. Create a test project in your cluster

```bash
oc new-project playground
```

2. Create a privileged service account

```bash
# Note: generate-index-sa is defaulted within the trigger template
oc create sa generate-index-sa
oc adm policy add-scc-to-user privileged -z generate-index-sa
```

3. Clone this repository

```bash
git clone https://github.com/amisstea/operator-pipelines-poc.git
```

4. Apply the OpenShift resources

```bash
oc apply -R -f operator-pipelines-poc
```

5. [Optional] Expose your local CRC cluster to the internet, if necessary.
   Tools such as [ngrok](https://dashboard.ngrok.com/get-started/setup) are
   handy for this. This is only required if your cluster cannot be reached
   from beyond your network. For example:

```bash
# The CRC route can be discovered by running `oc get routes`
ngrok http --host-header=rewrite el-operator-cert-github-listener-playground.apps-crc.testing:80
```

6. Fork the [community-operators](https://github.com/operator-framework/community-operators) repo

7. Setup a GitHub webhook to reach your local CRC cluster. Point
   this hook at your publicly accessible tunnel or OpenShift route. The only
   event type that is supported for now is `Pull requests`.

8. Submit a pull request from a branch in your forked community-operators repo
   into the `master` branch of the same fork. To be a little more
   representative of the branching decisions the production pipelines would
   need to make, the bundle included with the pull request MUST specify the
   `com.redhat.openshift.versions` annotation. An example can be found
   [here](https://github.com/amisstea/community-operators/pull/1).

That's it! The pipeline should begin running.

## Limitations
1. Tekton does not yet support a
   [Pipeline-in-pipeline](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md)
   feature. This makes composable pipelines difficult. Until this feature is
   readily available, pipeline definitions may be repetitive and not share the
   exact same implementation.
2. Some tasks in the pipeline require a privileged security context. The
   OpenShift Pipelines Operator supplies a privileged buildah `ClusterTask`
   so that isn't much of a concern. The index generation task, however, uses
   `opm` with `podman` under the hood. Running `podman` within a container on
   OpenShift still requires privileged access. This may be problematic in a
   shared cluster environment where administrators are reluctant to grant such
   a wide set of permissions.

## Outstanding PoC Work
1. Triggering a `PipelineRun` from a local development machine. This is needed to
   show feasibility of a partner running the validate, build and test-like tasks
   of the demo pipeline against an arbitrary bundle source directory.
2. Running tasks after a series of conditionally skipped Tekton branches
   (index generation/building) is [not as simple](https://github.com/tektoncd/community/blob/main/teps/0059-skipping-strategies.md)
   as one might expect. [Split-joining](https://github.com/tektoncd/pipeline/issues/3929)
   branches, a common CI/CD workflow, is not available in Tekton's beta state.
   The latter tasks in the pipeline will probably need a solution that's more
   flexible than using the `finally` definition. This may come at the cost of
   complexity.
