![Successful cert pipeline run](img/cert-pipelinerun-details.png)

![Successful test pipeline run](img/test-pipelinerun-details.png)

# operator-pipelines-poc
Proof of Concept for an Operator Certification Pipeline using OpenShift Pipelines.

This proof of concept was put together using Red Hat CodeReady Containers (CRC).
Alternatively, it should be possible to run these pipelines on any OpenShift
cluster of your choosing with minor adjustments.

Additionally, for demonstration purposes, this uses the already established
structure of the Community Operator Pipeline. It should ultimately be triggered
by pull requests made to certified and/or Red Hat Marketplace submission repos.

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
oc apply -R \
  -f operator-pipelines-poc/config \
  -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/github-set-status/0.2/github-set-status.yaml \
  -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/aws-cli/0.2/aws-cli.yaml
```

4. Fork the [community-operators](https://github.com/operator-framework/community-operators)
   repo. Modify one of the operators utilizing the bundle format
   ([example](https://github.com/amisstea/community-operators/commit/88c9c0e4e843e4f5fb34033abb924606017064aa))
   and push your local changes so they are accessible for testing.

## Running the Base Pipeline

Assuming the working directory is the root of this repo, the base pipeline
(the one a partner may run) can be manually triggered with a Git source
location. View all possible params by running `tkn pipeline describe operator-base-pipeline`.

```bash
tkn pipeline start operator-base-pipeline \
  --param git_repo_url=[GIT_REPO] \
  --param git_repo_name=[GIT_REPO_FULL_NAME] \
  --param git_revision=[BRANCH,COMMIT,TAG] \
  --param bundle_path=[RELATIVE_PATH_WITHIN_GIT_REPO] \
  --workspace name=pipeline,volumeClaimTemplateFile=test/workspace-template.yml \
  --showlog
```

Ex:

```bash
tkn pipeline start operator-test-pipeline \
  --param git_repo_url=https://github.com/amisstea/community-operators.git \
  --param git_repo_name=amisstea/community-operators \
  --param git_revision=test-branch \
  --param bundle_path=community-operators/kogito-operator/1.6.0 \
  --workspace name=pipeline,volumeClaimTemplateFile=test/workspace-template.yml \
  --showlog
```

That's it! A `PipelineRun` should now be running.

## Running the Red Hat Certification Pipeline

The Red Hat certification pipeline is meant to be triggered by a GitHub pull
request. By default it will always upload test results to AWS S3 as well as
create status checks for the commit in GitHub.

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

3. Create an AWS secret containing credentials and a config. The default secret
   name is `aws-secret`. It may look something like the following:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-secret
type: Opaque
stringData:
  credentials: |-
    [default]
    aws_access_key_id     = [ACCESS_KEY_ID]
    aws_secret_access_key = [SECRET_ACCESS_KEY]
  config: |-
    [default]
    region = us-east-1
```

Apply this secret.
```bash
oc apply -f aws-secret.yml
```

4. Create a github secret. The pipeline currently assumes the default secret
   name `github` with a `token` key.

```bash
oc create secret generic github --from-literal token="[TOKEN]"
```

5. Submit a pull request from your branch to the `master` branch of your
   forked `community-operators` repo. This should trigger the creation of a
   `PipelineRun`.

## Limitations

### Pipelines in Pipelines

Tekton does not yet support
[pipelines-in-pipelines](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md).

#### Workaround

The `tkn` `ClusterTask` can be used to start pipelines and monitor their logs.
An example of this can be found in the `run-operator-test-pipeline` pipeline
task. A means for propagating results from an embedded pipeline into a pipeline
which wraps it may need more research. It may need to be solved with a more
advanced feature like `CustomTasks` which is still in alpha and a
[non-trivial](https://github.com/tektoncd/community/blob/main/teps/0002-custom-tasks.md#drawbacks)
amount of effort to implement.

### Running Tasks After Skipped Tasks
Running a task after a conditional task
(index generation/building/testing) is [not as simple](https://github.com/tektoncd/community/blob/main/teps/0059-skipping-strategies.md)
as one might expect. [Split-joining](https://github.com/tektoncd/pipeline/issues/3929)
conditional branches, a common CI/CD workflow, is not available in Tekton's beta state.
The `Pipeline` schema has a `finally` section for tasks to always run at the end
but that won't be ideal in a lot of circumstances. On top of that, tasks under
`finally` are always run in parallel.

#### Workaround

Embedding pipelines in pipelines can likely solve most problems at the expense
of creating additional `PipelineRuns` and the confusion that may cause. See the
workaround.

Alternatively, where possible, tasks may need to be constructed such that they
always pass but intentionally behave as a no-op given some params.
