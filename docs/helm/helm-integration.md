# Using Flux with Helm

You can release charts to your cluster via "GitOps", by combining Flux
and the Flux Helm Operator (also in
[weaveworks/flux](https://github.com/weaveworks/flux)).

The essential mechanism is this: the declaration of a Helm release is
represented by a custom resource, specifying the chart and its
values. If you put such a resource in your git repo as a file, Flux
will apply it to the cluster, and once it's in the cluster, the Helm
Operator will make sure the release exists by installing or upgrading
it.

## The `HelmRelease` custom resource

Each release of a chart is declared by a `HelmRelease`
resource. The schema for these resources is given in [the custom
resource definition](https://github.com/weaveworks/flux/blob/master/deploy-helm/flux-helm-release-crd.yaml). They
look like this:

```yaml
---
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: rabbit
  namespace: default
spec:
  releaseName: rabbitmq
  chart:
    repository: https://kubernetes-charts.storage.googleapis.com/
    name: rabbitmq
    version: 3.3.6
  values:
    replicas: 1
```

The `releaseName` will be given to Helm as the release name. If not
supplied, it will be generated by affixing the namespace to the
resource name. In the above example, if `releaseName` were not given,
it would be generated as `default-rabbitmq`. Because of the way Helm
works, release names must be unique in the cluster.

The `chart` section gives a pointer to the chart; in this case, to a
chart in a Helm repo. Since the helm operator is running in your
cluster, and doesn't have access to local configuration, the
repository is given as a URL rather than an alias (the URL in the
example is what's usually aliased as `stable`). The `name` and
`version` specify the chart to release.

The `values` section is where you provide the value overrides for the
chart. This is as you would put in a `values.yaml` file, but inlined
into the structure of the resource. See below for examples.

### Using a chart from a Git repo instead of a Helm repo

You can refer to a chart from a _git_ repo, rather than a chart repo,
with a `chart:` section like this:

```yaml
spec:
  chart:
    git: git@github.com:weaveworks/flux-get-started
    ref: master
    path: charts/ghost
```

In this case, the git repo will be cloned, and the chart will be
released from the ref given (which defaults to `master`, if not
supplied). Commits to the git repo may result in releases, if they
update the chart at the path given.

Note that you will usually need to provide an SSH key to grant access
to the git repository. The example deployment shows how to mount a
secret at the expected location of the key (`/etc/fluxd/ssh/`). If you
need more than one SSH key, you'll need to also mount an adapted
ssh_config; this is also demonstrated in the example deployment.

#### Notifying Helm Operator about Git changes

The Helm Operator fetches the upstream of mirrored Git repositories
with a 5 minute interval. In some scenarios (think CI/CD), you may not
want to wait for this interval to occur.

To help you with this the Helm Operator serves a HTTP API endpoint to
instruct it to immediately refresh all Git mirrors.

```sh
$ kubectl -n flux port-forward deployment/flux-helm-operator 3030:3030 &
$ curl -XPOST http://localhost:3030/api/v1/sync-git
OK
```

> **Note:** the HTTP API has no built-in authentication, this means you
> either need to port forward before making the request or put something
> in front of it to serve as a gatekeeper.

### Reinstalling a Helm release

If a Helm release upgrade fails due to incompatible changes like modifying
an immutable field (e.g. headless svc to ClusterIP)  
you can reinstall it using the following command:

```sh
$ kubectl delete hr/my-release
```

When the Helm Operator receives a delete event from Kubernetes API it will
call Tiller and purge the Helm release. On the next Flux sync, the Helm Release
object will be created and the Helm Operator will install it.

### What the Helm Operator does

When the Helm Operator sees a `HelmRelease` resource in the
cluster, it either installs or upgrades the named Helm release so that
the chart is released as specified.

It will also notice when a `HelmRelease` resource is updated, and
take action accordingly.

## Supplying values to the chart

You can supply values to be used with the chart when installing it, in
two ways.

### `.spec.values`

This is a YAML map as you'd put in a file and supply to Helm with `-f
values.yaml`, but inlined into the `HelmRelease` manifest. For
example,

```yaml
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
# metadata: ...
spec:
  # chart: ...
  values:
    foo: value1
    bar:
    baz: value2
    oof:
    - item1
    - item2
```

### `.spec.valuesFrom`

This is a list of secrets, config maps (in the same namespace as the
`HelmRelease`) or external sources (URLs) from which to take values.

The values are merged in the order given, with later values
overwriting earlier. These values always have a lower priority than
those passed via the `.spec.values` parameter.

This is useful if you want to have defaults such as the `region`,
`clustername`, `environment`, a local docker registry URL, etc., or if
you simply want to have values not checked into git as plaintext.

#### Config maps

```yaml
spec:
  # chart: ...
  valuesFrom:
  - configMapKeyRef:
      # Name of the config map, must be in the same namespace as the
      # HelmRelease
      name: default-values  # mandatory
      # Key in the config map to get the values from
      key: values.yaml      # optional; defaults to values.yaml
      # If set to true successful retrieval of the values file is no
      # longer mandatory
      optional: false       # optional; defaults to false
```

#### Secrets

```yaml
spec:
  # chart: ...
  valuesFrom:
  - secretKeyRef:
      # Name of the secret, must be in the same namespace as the
      # HelmRelease
      name: default-values # mandatory
      # Key in the secret to get thre values from
      key: values.yaml     # optional; defaults to values.yaml
      # If set to true successful retrieval of the values file is no
      # longer mandatory
      optional: true       # optional; defaults to false
```

#### External sources

```yaml
spec:
  # chart: ...
  valuesFrom:
  - externalSourceRef:
      # URL of the values.yaml
      url: https://example.com/static/raw/values.yaml # mandatory
      # If set to true successful retrieval of the values file is no
      # longer mandatory
      optional: true                                       # optional; defaults to false
```

#### Chart files

```yaml
spec:
  # chart: ...
  valuesFrom:
  - chartFileRef:
      # path within the helm chart (from git repo) where environment-prod.yaml is located
      path: overrides/environment-prod.yaml # mandatory
      # If set to true successful retrieval of the values file is no
      # longer mandatory
      optional: true                                       # optional; defaults to false
```

## Upgrading images in a `HelmRelease` using Flux

If the chart you're using in a `HelmRelease` lets you specify the
particular images to run, you will usually be able to update them with
Flux, the same way you can with Deployments and so on.

> **Note:** for automation to work, the repository _and_ tag should be
> defined, as Flux determines image updates based on what it reads in
> the `.spec.values` of the `HelmRelease`.

Flux interprets certain commonly used structures in the `values`
section of a `HelmRelease` as referring to images. The following
are understood (showing just the `values` section):

```yaml
values:
  image: repo/image:version
```

```yaml
values:
  image: repo/image
  tag: version
```

```yaml
values:
  registry: docker.io
  image: repo/image
  tag: version
```

```yaml
values:
  image:
    repository: repo/image
    tag: version
```

```yaml
values:
  image:
    registry: docker.io
    repository: repo/image
    tag: version
```

These can appear at the top level (immediately under `values:`), or in
a subsection (under a key, itself under `values:`). Other values
may be mixed in arbitrarily. Here's an example of a values section
that specifies two images, along with some other configuration:

```yaml
values:
  persistent: true

  # image that will be labeled "chart-image"
  image: repo/image1:version

  subsystem:
    # image that will be labeled "subsystem"
    image:
      repository: repo/image2
      tag: version
      imagePullPolicy: IfNotPresent
    port: 4040
```

You can use the [same annotations](../using/fluxctl.md) in
the `HelmRelease` as you would for a Deployment or other workload,
to control updates and automation. For the purpose of specifying
filters, the container name is either `chart-image` (if at the top
level), or the key under which the image is given (e.g., `"subsystem"`
from the example above).

Top level image example:

```yaml
kind: HelmRelease
metadata:
  annotations:
    flux.weave.works/automated: "true"
    flux.weave.works/tag.chart-image: semver:~4.0
spec:
  values:
    image:
      repository: bitnami/mongodb
      tag: 4.0.3
```

Sub-section images example:

```yaml
kind: HelmRelease
metadata:
  annotations:
    flux.weave.works/automated: "true"
    flux.weave.works/tag.prometheus: semver:~2.3
    flux.weave.works/tag.alertmanager: glob:v0.15.*
    flux.weave.works/tag.nats: regex:^0.6.*
spec:
  values:
    prometheus:
      image: prom/prometheus:v2.3.1
    alertmanager:
      image: prom/alertmanager:v0.15.0
    nats:
      image:
        repository: nats-streaming
        tag: 0.6.0
```

-------------

<a name="why-repo-urls">**Why use URLs to refer to repositories, rather than names?**</a> [^](#cite-why-repo-urls)

A `HelmRelease` must be able to stand on its own. If we used names
in the spec, which were resolved to URLs elsewhere (e.g., in a
`repositories.yaml` supplied to the operator), it would be possible to
change the meaning of a `HelmRelease` without altering it. This is
undesirable because it makes it hard to specify exactly what you want,
in the one place; or to read exactly what is being specified, in the
one place. In other words, it's better to be explicit.

## Rollbacks

From time to time a release made by the Helm operator may fail, it is
possible to automate the rollback of a failed release by setting
`.spec.rollback.enable` to `true` on the `HelmRelease` resource.

> **Note:** a successful rollback of a Helm chart containing a 
> `StatefulSet` resource is known to be tricky, and one of the main
> reasons automated rollbacks are not enabled by default for all
> `HelmRelease`s. Verify a manual rollback of your Helm chart does
> not cause any problems before enabling it.

When enabled, the Helm operator will detect a faulty upgrade and
perform a rollback, it will not attempt a new upgrade unless it
detects a change in values and/or the chart.

### Configuration

```yaml
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
# metadata: ...
spec:
  # Listed values are the defaults.
  rollback:
    # If set, will perform rollbacks for this release.
    enable: false
    # If set, will force resource update through delete/recreate if
    # needed.
    force: false
    # Prevent hooks from running during rollback.
    disableHooks: false
    # Time in seconds to wait for any individual Kubernetes operation.
    timeout: 300
    # If set, will wait until all Pods, PVCs, Services, and minimum
    # number of Pods of a Deployment are in a ready state before
    # marking the release as successful. It will wait for as long
    # as the set timeout.
    wait: false
```

## Authentication

At present, per-resource authentication is not implemented. The
`HelmRelease` definition includes a field `chartPullSecret` for
attaching a `repositories.yaml` file, but this is ignored for now.

Instead, you need to provide the operator with credentials and keys (see the
following [Authentication for Helm repos](#authentication-for-helm-repos)
section for how to do this).

### Authentication for Helm repos

As a workaround, you can mount a `repositories.yaml` file with
authentication already configured, into the operator container.

> **Note:** When using a custom `repositories.yaml` the
[default](https://github.com/weaveworks/flux/blob/master/docker/helm-repositories.yaml)
that ships with the operator is overwritten. This means that for any
repository you want to make use of you should manually add an entry to
your `repositories.yaml` file.

To prepare a file, add the repo _locally_ as you would normally:

```sh
helm repo add <URL> --username <username> --password <password>
```

You need to doctor this file a little, since it will likely contain
absolute paths that will be wrong when mounted inside the
container. Copy the file and replace all the `cache` entries with just
the filename.

```sh
cp ~/.helm/repository/repositories.yaml .
sed -i -e 's/^\( *cache: \).*\/\(.*\.yaml\)/\1\2/g' repositories.yaml
```

Now you can create a secret in the same namespace as you're running
the Helm operator, from the repositories file:

```sh
kubectl create secret generic flux-helm-repositories --from-file=./repositories.yaml
```

Lastly, mount that secret into the container. This can be done by
setting `helmOperator.configureRepositories.enable` to `true` for the
flux Helm release, or as shown in the commented-out sections of the
[example deployment](https://github.com/weaveworks/flux/blob/master/deploy-helm/helm-operator-deployment.yaml).

#### Azure ACR repositories

For Azure ACR repositories, the entry in `repositories.yaml` created by
running `az acr helm repo add` is unsufficient for the Helm operator.
Instead you will need to [create a service principal](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal#create-a-service-principal)
and use the plain text id and password this gives you. For example:

```yaml
- caFile: ""
  cache: <repository>-index.yaml
  certFile: ""
  keyFile: ""
  name: <repository>
  url: https://<repository>.azurecr.io/helm/v1/repo
  username: <service principal id>
  password: <service principal password>
```

### Authentication for Git repos

In general, it's necessary to have an SSH key to clone a git
repo. This is sometimes (e.g., on GitHub) called a "deploy key". To
use a chart from git, the Helm Operator needs a key with read-only
access.

To provide an SSH key, put the key in a secret under the entry
`identity`, and mount it into the operator container as shown in the
[example deployment](https://github.com/weaveworks/flux/blob/master/deploy-helm/helm-operator-deployment.yaml).
The default ssh_config expects an identity file at
`/etc/fluxd/ssh/identity`, which is where it'll be if you just
uncomment the blocks from the example.

If you're using more than one repository, you may need to provide more
than one SSH key. In that case, you can create a secret with an entry
for each key, and mount that _as well as_ an ssh_config file
mentioning each key as an `IdentityFile`.
