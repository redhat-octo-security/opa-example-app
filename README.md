# opa-example-app

OPA Example App

## Install Dependencies

Before proceeding with this tutorial, you'll need to install OpenShift
Pipelines by installing the OpenShift Pipelines Operator:
1. You can view the instructions to install the OpenShift Pipelines Operator
   [here](https://github.com/openshift/tektoncd-pipeline-operator) or if you
   have an OpenShift cluster running, navigate to the Operators drop-down menu
   on the left and click on "OperatorHub". Then search for OpenShift Pipelines
   Operator and install it.
1. Install OPA Gatekeeper by following these
   [instructions](https://github.com/open-policy-agent/gatekeeper#installation).
   This tutorial was constructed using OPA Gatekeeper `v3.1.0-beta.9`. Once
   installed, OPA Gatekeeper specifies a seccomp profile that on OpenShift
   requires the pods to run as privileged so you'll need to make sure to add
   the `privileged` security context constraint to the
   `gatekeeper-admin` service account by running:
   ```bash
   oc -n gatekeeper-system adm policy add-scc-to-user privileged -z gatekeeper-admin
   ```
1. Install [podman](https://podman.io/).

## Fork This Repository

You'll want to fork this repository in order run through the tutorial so that
you can commit and push changes to trigger builds.

## Configure the cluster

- Create the Namespace where the resoures will live:

```bash
oc create namespace opa-example-app
oc create namespace opa-example-app-trigger
```

- Set the namespace for the `current-context`:

```bash
oc project opa-example-app-trigger
```

- Create the secret to access your container registry. If using Quay, you can
  create a robot account and provide it the necessary permissions to push to
  your container registry repo.

```bash
oc create secret docker-registry regcred \
                    --docker-server=<your-registry-server> \
                    --docker-username=<your-name> \
                    --docker-password=<your-pword> \
                    --docker-email=<your-email>
```

- Create the trigger admin service account, role and rolebinding

```bash
oc apply -f ./config/tekton/trigger/admin-role.yaml
```

- Create the webhook user, role and rolebinding

```bash
oc apply -f ./config/tekton/trigger/webhook-role.yaml
```

- Create the app deploy role and rolebinding in the namespace that will host
  the opa-example-app:

```bash
oc -n opa-example-app apply -f ./config/tekton/trigger/app-role.yaml
```

## Install the Pipeline and Trigger

### Install the Pipeline

To install the pipeline run:

```bash
oc apply -f ./config/tekton/trigger/pipeline.yaml
```

### Install the TriggerTemplate, TriggerBinding and EventListener

Be sure to replace the image `PipelineResource` `image-source` `url` field with
the respective container registry and repository to use for pushing the built
image. Then run:

```bash
oc apply -f ./config/tekton/trigger/triggers.yaml
```

## Add Ingress and GitHub-Webhook Tasks

```bash
oc apply -f ./config/tekton/trigger/create-ingress.yaml
oc apply -f ./config/tekton/trigger/create-webhook.yaml
```

## Run Ingress Task

Be sure to replace the `ExternalDomain` parameter value with your FQDN. This
will be used by the GitHub webhook to reach the ingress in your cluster in
order to pass the relevent GitHub commit details to the `EventListener` service
running in your cluster. Then run:

```bash
oc apply -f ./config/tekton/trigger/ingress-run.yaml
```

## Run GitHub Webhook Task

You will need to create a [GitHub Personal Access
Token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line#creating-a-token)
with the following access:

- public_repo
- admin:repo_hook

Next, create a secret like so with your access token.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webhook-secret
    namespace: opa-example-app
    stringData:
      token: YOUR-GITHUB-ACCESS-TOKEN
        secret: random-string-data
```

Next you'll want to edit the `webhook-run.yaml` file:
- Modify the `GitHubOrg` and `GitHubUser` fields to match your setup.
- Modify the `ExternalDomain` field to match the FQDN used in
  `ingress-run.yaml` for configuring the GitHub webhook to use this FQDN to
  talk to the `EventListener`.

Then Create the webhook task:

```bash
oc apply -f ./config/tekton/trigger/webhook-run.yaml
```

## Watch the Trigger and Pipeline Work!

Commit and push an empty commit to your development repo.

```bash
git commit -a -m "build commit" --allow-empty && git push origin mybranch
```

## Install OPA Gatekeeper ConstraintTemplate and Constraint

First we'll install the `ConstraintTemplate` and `K8sTrustedRegistries`
constraint to prevent untrusted registries from being used.

```bash
oc apply -f ./config/opa/trustedregistries-template.yaml
oc apply -f ./config/opa/trustedregistries.yaml
```

## Update Deployment Image Registry

In order to exercise the OPA policy, we'll need to attempt to use an untrusted
registry:

```bash
sed -i s/quay.io/gcr.io/ ./config/ocp/deployment.yaml
```

Commit and push the changes to watch OPA prevent the deployment.

## Giving Developers Feedback Sooner (Shift Left)

Wouldn't it be great if developers didn't have to wait for the entire CI/CD
pipeline to complete only to realize the operation they want to perform isn't
allowed? To give developers feedback sooner, we need to move the policy
prevention mechanisms earlier in the develoment lifecycle.

### Add Early Step to Pipeline

One way to achieve this, is to add an initial step in the pipeline that
attempts to perform the `oc apply` operations with the `--server-dry-run`
flag so that we can get feedback sooner on whether the policy prevents this
operation. To do this, let's add a task and update the pipeline steps to
include a dry run step prior to anything else being run in the pipeline. Go
ahead and apply these changes:

```bash
oc apply -f ./config/tekton/trigger/pipeline-opa.yaml
```

Commit and push an empty commit to your development repo.

```bash
git commit -a -m "build commit" --allow-empty && git push origin mybranch
```

And watch the pipeline fail earlier from the `quay.io` to `gcr.io` change.

### Add Earlier Step to Development Workflow

We can give developers feedback even earlier in the developer lifecycle so that
the developer doesn't even have to kick off the CI/CD pipeline and consume
existing resources. In order to do that, we'll use a tool called
[`conftest`](https://github.com/instrumenta/conftest). It is a tool that
facilitates testing your configuration files against OPA. In our case, we want
to test the trusted registries Rego policy against our `deployment.yaml` file.

Using `conftest`, we can also leverage git `pre-commit` hooks to add a policy
check. In order to do that, create a symbolic link to the git `pre-commit` hook
script:

```bash
ln -rs hooks/pre-commit.sh .git/hooks/pre-commit
```

Now revert the previous `gcr.io` registry change and then attempt to use an
untrusted registry again:

```bash
sed -i s/quay.io/gcr.io/ ./config/ocp/deployment.yaml
```

Attempt to commit and you will receive the same error immediately instead of
waiting for the CI/CD pipeline to complete!

## Cleanup

Delete the namespaces:

```bash
oc delete namespace opa-example-app
oc delete namespace opa-example-app-trigger
```
