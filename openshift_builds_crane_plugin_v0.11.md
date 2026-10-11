# Migrate BuildConfigs to Builds for Red Hat OpenShift with the crane plugin

**The BuildConfig migration tool now ships as a crane plugin, installed from the crane plugin index**

In January we announced a developer preview of BuildConfig migration in Crane, in
[Migrate BuildConfig resources to Builds for Red Hat OpenShift with Crane](https://developers.redhat.com/articles/2026/01/27/migrate-buildconfig-resources-builds-red-hat-openshift-crane).
That post showed how to turn an OpenShift BuildConfig into a Shipwright Build, the resource
that Builds for Red Hat OpenShift runs.

The conversion code has since moved out of Crane into its own plugin,
[crane-plugin-buildconfig-to-builds](https://github.com/migtools/crane-plugin-buildconfig-to-builds).
Version [v0.11.0](https://github.com/migtools/crane-plugin-buildconfig-to-builds/releases/tag/v0.11.0)
is its first release. Crane installs it from its plugin index with one command.

This post covers what changed since January, then walks one BuildConfig through the new flow
until it builds an image on the target cluster.

## What changed since January

In January you ran `crane convert`. The conversion now runs as a
transform plugin inside Crane's usual export, transform and apply flow. That is the same flow
Crane uses for the rest of a namespace, so the BuildConfigs move together with the Secrets,
ConfigMaps and ServiceAccounts they need.

You no longer need a download script.
`crane plugin-manager add BuildConfigToBuildsPlugin` fetches the released binary for your
platform.

The plugin reads the exported files and writes new ones. It never
contacts a cluster, so you can read every converted Build before anything reaches the target.

When the plugin drops or changes a BuildConfig field, it adds
a warning to the converted Build, in the `crane.konveyor.io/conversion-warnings` annotation.
The same text goes to the log. Each warning has a numbered entry in the
[support matrix](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/support-matrix.md)
that says what happened and what to do by hand.

v0.11.0 handles Git, binary and image sources. It also
carries build args, environment variables, volumes, the builder image, output image labels,
resource requests and limits, node selectors, timeouts and build history limits.

A BuildConfig the plugin cannot convert does not stop the run. It stays in the output unchanged,
with two annotations that say it was skipped or failed and why.

## What gets converted

| BuildConfig strategy | Builds for Red Hat OpenShift strategy | Outcome |
|---|---|---|
| Docker | `buildah` | converted |
| Source (S2I) | `source-to-image` | converted |
| Custom | none | skipped, left unchanged with annotations |
| JenkinsPipeline | none | skipped, left unchanged with annotations |

The two strategy names are the ones the Builds for Red Hat OpenShift operator installs.

## Prerequisites

- A source cluster with the BuildConfigs you want to move, and `oc` access to it.
- A target cluster with the Builds for Red Hat OpenShift operator installed.
- Crane v0.11.0 or newer. Crane v0.11.0-alpha.1 also works. Earlier versions produce no Build.

## Example: a Docker build with a Git source

This is the same Ruby application as in January. The BuildConfig builds
`ruby-hello-world` from its own Dockerfile and pushes the image to an ImageStream. On the
target cluster the image goes to an external registry instead.

### Step 1: Look at the BuildConfig

On the source cluster, in namespace `buildconfig-test`:

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ruby-hello-world-docker
  namespace: buildconfig-test
  labels:
    app: ruby-hello-world
    build-type: docker
spec:
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
  source:
    type: Git
    git:
      uri: https://github.com/openshift/ruby-hello-world.git
      ref: master
  output:
    to:
      kind: ImageStreamTag
      name: ruby-hello-world:latest
    pushSecret:
      name: quay-push
  triggers:
    - type: ConfigChange
```

`quay-push` is a Secret in the same namespace that holds the credential for `quay.io`.

### Step 2: Install Crane

Download the binary for your platform from the
[Crane releases page](https://github.com/migtools/crane/releases). On Linux on x86:

```bash
CRANE_VERSION=v0.11.0
curl -Lo crane "https://github.com/migtools/crane/releases/download/${CRANE_VERSION}/crane_linux_amd64"
chmod +x crane
sudo mv crane /usr/local/bin/
crane version
```

The other assets follow the pattern `crane_<os>_<arch>`, for example `crane_darwin_arm64`.

### Step 3: Install the plugin

```bash
crane plugin-manager add BuildConfigToBuildsPlugin
crane plugin-manager list
```

The binary goes to `$HOME/.local/share/crane/plugins`, where `crane transform` finds it.

### Step 4: Export, transform and write the output

Log in to the source cluster, then run:

```bash
crane export -n buildconfig-test

crane transform KubernetesPlugin BuildConfigToBuildsPlugin \
  --optional-flags '{"registry-mapping":"image-registry.openshift-image-registry.svc:5000=quay.io/myorg"}'

crane apply
```

`crane export` saves every resource in the namespace to local files.

`crane transform` runs two plugins. `KubernetesPlugin` is built into Crane and removes fields
such as `uid` and `status` that would stop a resource applying to another cluster.
`BuildConfigToBuildsPlugin` converts the BuildConfigs.

The `registry-mapping` flag moves the output image off the internal OpenShift registry, which
may not exist on the target, and onto `quay.io/myorg`. Replace `myorg` with your organization.

`crane apply` writes the result to `output/`. It does not touch a cluster.

### Step 5: Read the Build and its warnings

The Build is in
`output/resources/buildconfig-test/Build_shipwright.io_v1beta1_buildconfig-test_ruby-hello-world-docker.yaml`:

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: ruby-hello-world-docker
  namespace: buildconfig-test
  annotations:
    buildconfig-to-shipwright/conversion-outcome: converted-with-warnings
    buildconfig-to-shipwright/original-triggers: '[{"type":"ConfigChange"}]'
    crane.konveyor.io/conversion-warnings: |-
      ...
spec:
  output:
    image: quay.io/myorg/buildconfig-test/ruby-hello-world:latest
    pushSecret: quay-push
  paramValues:
  - name: dockerfile
    value: Dockerfile
  retention:
    failedLimit: 5
    succeededLimit: 5
  source:
    type: Git
    git:
      url: https://github.com/openshift/ruby-hello-world.git
      revision: master
  strategy:
    kind: ClusterBuildStrategy
    name: buildah
```

The Docker strategy became the `buildah` strategy. The Dockerfile path became a parameter. The
Git source, the push secret and the labels carried over. The ImageStream output became a
registry address.

The `conversion-warnings` annotation lists four things this BuildConfig lost:

- The image now goes to Quay, so the ImageStream on the source cluster stops updating. A
  Deployment that rolled out on that ImageStream needs the new image address.
- The `Serial` run policy is gone. BuildRuns, the objects that run a Build, run at the same time.
- The `ConfigChange` trigger is gone, so the first build does not start on its own.
- The trigger summary: no trigger works in Shipwright today.

Read these before you apply. They are the to-do list for this Build.

### Step 6: Apply to the target cluster and run the build

Log in to the target cluster:

```bash
oc apply -R -f output/resources/

oc wait --for=jsonpath='{.status.registered}'=True \
  build.shipwright.io/ruby-hello-world-docker -n buildconfig-test --timeout=120s
```

Write `build.shipwright.io`, not `build`. On OpenShift the short name points at the OpenShift
Build API.

The apply may report an error for an `ImageTag` resource: `spec is a required field during
creation`. Crane exports it along with the ImageStream, and the API refuses it on create. The
Build does not use it, so you can ignore the error or delete that file before you apply.

The trigger is gone, so start the first build yourself:

```bash
cat <<'EOF' | oc create -f -
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  generateName: ruby-hello-world-docker-
  namespace: buildconfig-test
spec:
  build:
    name: ruby-hello-world-docker
EOF

oc get buildrun -n buildconfig-test -w
```

The BuildRun runs as the `pipeline` service account, which the operator sets up. When it
reaches `Succeeded`, the image is in `quay.io/myorg/buildconfig-test/ruby-hello-world:latest`.

The run takes about a minute:

```
NAME                            SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
ruby-hello-world-docker-6bddt   True        Succeeded   62s         1s
```

## What the plugin cannot convert

Some BuildConfig features have no equivalent in Builds for Red Hat OpenShift today. The main
ones:

- Custom and JenkinsPipeline strategies are skipped. Move a Jenkins pipeline to
  OpenShift Pipelines.
- Webhook, ImageChange and ConfigChange triggers are dropped. The plugin keeps
  them in an annotation. The
  [trigger migration guide](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/trigger-migration.md)
  shows how to start builds from a webhook listener, a Pipeline or a CronJob.
- Post-commit hooks are dropped. Run your tests as a Pipeline step after the build.
- Binary builds convert, but each build starts with `shp build upload` instead of
  `oc start-build --from-dir`.
- Secrets and ConfigMaps used as build input need a copy of the build strategy with
  an extra volume. The
  [volume migration guide](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/volume-migration.md)
  has the steps.

The full list, with a workaround for each item and what is planned, is on the
[known limitations](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/known-limitations.md)
page.

## More examples

The plugin repository has
[four worked examples](https://github.com/migtools/crane-plugin-buildconfig-to-builds/tree/v0.11.0/docs/examples).
Each one has the input BuildConfig, the flags, the exact output and the steps on the target
cluster. They cover an S2I build whose builder image is in an ImageStream, a binary build fed
by a directory upload, and a Docker build that loses several fields. A test checks their
output on every change to the plugin, so they stay correct.

## Current availability and support

The plugin is an upstream release in the Konveyor Crane project. It is not yet part of a
supported Red Hat product. We are working toward general availability and want to hear how it
handles your BuildConfigs.

Open an issue on the
[plugin repository](https://github.com/migtools/crane-plugin-buildconfig-to-builds/issues)
with the BuildConfig, with names and secrets removed, and the warnings you got.

## Resources

- [January post: Migrate BuildConfig resources to Builds for Red Hat OpenShift with Crane](https://developers.redhat.com/articles/2026/01/27/migrate-buildconfig-resources-builds-red-hat-openshift-crane)
- [Plugin v0.11.0 release notes](https://github.com/migtools/crane-plugin-buildconfig-to-builds/releases/tag/v0.11.0)
- [Plugin README](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/README.md)
- [Support matrix: every BuildConfig field](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/support-matrix.md)
- [Known limitations](https://github.com/migtools/crane-plugin-buildconfig-to-builds/blob/v0.11.0/docs/known-limitations.md)
- [Crane](https://github.com/migtools/crane)
- [Shipwright](https://shipwright.io/)
