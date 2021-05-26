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

2. Clone this repository

```bash
git clone https://github.com/amisstea/operator-pipelines-poc.git
```

3. Apply the OpenShift resources

```bash
oc apply -R -f operator-pipelines-poc
```

4. [Optional] Expose your local CRC cluster to the internet, if necessary.
   Tools such as [ngrok](https://dashboard.ngrok.com/get-started/setup) are
   handy for this. This is only required if your cluster cannot be reached
   from beyond your network. For example:

```bash
OCP_ROUTE=$(oc get route operator-cert-ci -o jsonpath='{.spec.host}')
ngrok http --host-header=rewrite $OCP_ROUTE:80
```

5. Fork the [community-operators](https://github.com/operator-framework/community-operators) repo

6. Setup a GitHub webhook to reach your local CRC cluster. Point
   this hook at your publicly accessible tunnel or OpenShift route. The only
   event type that is supported for now is `Pull requests`.

7. Submit a pull request from a branch in your forked community-operators repo
   into the `master` branch of the same fork. To be a little more
   representative of the branching decisions the production pipelines would
   need to make, the bundle included with the pull request MUST specify the
   `com.redhat.openshift.versions` annotation. An example can be found
   [here](https://github.com/amisstea/community-operators/pull/1).

That's it! The pipeline should begin running.

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
1. Triggering a `PipelineRun` from a local development machine. This is needed to
   show feasibility of a partner running the validate, build and test-like tasks
   of the demo pipeline against an arbitrary bundle source directory.
2. Find a workaround to split-joining with conditional branches
