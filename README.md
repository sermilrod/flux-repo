# flux-repo

This is a sample repo on how to configure flux and the configuration repo to manage services through a GitOps pattern.

## Install flux in your cluster

Flux will be installed using helm so the first thing we need to do is install helm and create the tiller service accout:
```
$ kubectl create ns flux
$ kubectl create sa tiller -n flux
$ helm init --service-account tiller --override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}' --tiller-namespace flux
```

This flux installation is a cluster-wide privileged installation as we want to leverage a single flux instance to deploy to multiple namespaces. Flux helm operator will use the tiller in this namespace to sync and deploy our services. Therefore, the tiller service account has to be a cluster-admin:
```
---
$ cat tiller-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: flux

$ kubectl apply -f tiller-clusterrolebinding.yaml
```

Now that tiller is ready we can deploy flux using the helm chart:
```
$ helm repo add weaveworks https://weaveworks.github.io/flux
$ helm repo update
$ helm upgrade -i flux \
    --set helmOperator.create=true \
    --set helmOperator.createCRD=true \
    --set git.url=git@github.com:sermilrod/flux-repo.git \
    --set helmOperator.tillerNamespace=flux \
    --set helmOperator.chartSyncInterval=1m \
    --set git.branch=master \
    --namespace flux \
    --tiller-namespace flux \
    weaveworks/flux
```

The previous helm chart will configure flux to sync `github.com:sermilrod/flux-repo` which is our configuration repo.
In addition it will only fetch changes from `master` branch and sets an unique label for your `prod-cluster-name`.

## Allow write access to the configuration repo

To get all the automated features from Flux, Flux will need access to your git repository to update configuration if necessary. You will need to add a deploy key to your configuration repository, in this case this repository.

Once Flux is deployed and all its pods healthy, you can fetch the deployment key as follows:
```
$ kubectl -n sergio-pipeline logs deployment/flux | grep identity.pub | cut -d '"' -f2
```

Copy the result and add it to your repository.

## Multi environment workflow

Let's say that we have 3 clusters dev, preprod and prod. As this is GitOps the reality of each cluster will also live in the config repository. There are other ways to achieve the same result but we are using git branches to achieve this goal.

| GIT BRANCH | KUBERNETES CLUSTER   |
|------------|----------------------|
| master     | prod-cluster-name    |
| preprod    | preprod-cluster-name |
| dev        | dev-cluster-name     |

This way we can have different versions in different environments. Rollouts are triggered by commits and rollbacks can be triggered by reverts or new commits as well.

There are many ways of implementing Continous Delivery pipelines leveraging the use of Flux. A possible pipeline workflow can be:

In dev:
    docker build -> docker push -> update /releases in dev -> wait for rollout -> run tests

In preprod:
    update /releases in preprod -> wait for rollout -> run tests

In prod:
    update /releases in prod -> wait for rollout -> run tests

And obviously you can chain them up to execute one after the previous succeeds.

In addition aligning a kubernetes cluster with a stable one is just a matter of resetting the git branch to the stable branch.

To make this work we need a flux installation per cluster. For the dev cluster:
```
$ helm upgrade -i flux \
    --set helmOperator.create=true \
    --set helmOperator.createCRD=true \
    --set git.url=git@github.com:sermilrod/flux-repo.git \
    --set helmOperator.tillerNamespace=flux \
    --set helmOperator.chartSyncInterval=1m \
    --set git.branch=dev \
    --namespace flux \
    --tiller-namespace flux \
    weaveworks/flux
```

For the preprod cluster:
```
$ helm upgrade -i flux \
    --set helmOperator.create=true \
    --set helmOperator.createCRD=true \
    --set git.url=git@github.com:sermilrod/flux-repo.git \
    --set helmOperator.tillerNamespace=flux \
    --set helmOperator.chartSyncInterval=1m \
    --set git.branch=preprod \
    --namespace flux \
    --tiller-namespace flux \
    weaveworks/flux
```

Make sure you fetch the key and add it to your repository:
```
$ kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```

With this configuration based on branches and flux syncronising the branches we can consolidate in a declarative way the state of the cluster very easily.