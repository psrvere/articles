## Getting Started

Ready to try the Developer Preview? Here's how to get started:

### Prerequisites

1. OpenShift Pipelines installed on your cluster
2. OpenShift Builds installed on your cluster

---

## Example: Migration of BuildConfig with Source Strategy and Git Source

Let's walk through a complete migration example step by step.

### Step 1: Create the BuildConfig

Let's take an example of the following BuildConfig (`ruby_source_git_buildconfig.yaml`):

- **`spec.source`** is of Git type and defines a GitHub URI, branch and secret to access the repository. It also defines context directory, secrets and configMaps which are available in the builder pod during the build process.
- **`spec.strategy`** is of Source type and defines a `from` image which overrides the builder image in the project's Dockerfile. It also defines a pull secret to pull images from a private registry, flags (`incremental`, `forcePull`), volumes, environment variables.

> **Note:** The `git.uri` field (`https://github.com/psrvere/ruby-hello-world`) points to a sample source code repository for this example. The `quay.io/prathore/ruby-27:latest` image is a private registry that you will not have access to. Please substitute your own source repository and registry credentials to try this example.

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ruby-source-git
  namespace: default
spec:
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: ruby-source-git:latest
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    type: Git
    git:
      ref: interim
      uri: git@github.com:psrvere/ruby-hello-world.git
    contextDir: .
    sourceSecret:
      name: github-ssh-secret
    secrets:
      - secret:
          name: database-credentials
        destinationDir: injected-config/secrets
    configMaps:
      - configMap:
          name: app-config
        destinationDir: injected-config/configmaps
  strategy:
    type: Source
    sourceStrategy:
      pullSecret:
        name: registry-creds
      from:
        kind: DockerImage
        name: quay.io/prathore/ruby-27:latest
      env:
        - name: RACK_ENV
          value: production
      incremental: true
      forcePull: true
      volumes:
        - name: bundler-config
          source:
            type: Secret
            secret:
              secretName: bundler-secret
          mounts:
            - destinationPath: /workspace/Gemfile.lock
```

### Step 2: Validate the BuildConfig

Make sure the BuildConfig is valid and runs successfully:

```bash
# Ensure secrets and configmaps are available in the cluster

oc create imagestream ruby-source-git
oc apply -f ruby_source_git_buildconfig.yaml
oc start-build ruby-source-git
oc get build
```

### Step 3: Download Crane CLI

Download the latest Crane CLI tool by running the following command:

```bash
curl -fsSL https://raw.githubusercontent.com/psrvere/download-crane/main/download-crane.sh | bash
```

This script will automatically detect your platform and download the appropriate Crane binary.

Alternatively, you can manually download the binary for your platform from the [Crane Releases](https://github.com/migtools/crane/releases/) page.

### Step 4: Run the Migration

Run the migration with the following command:

```bash
crane convert -r BuildConfigs -n default
```

**Flags:**
- `-r BuildConfigs` — Required for BuildConfig migration
- `-n default` — Specifies the namespace (mandatory)

The default behavior of the Crane CLI tool is to find all BuildConfigs in the provided namespace and migrate them to OpenShift Builds format.

**Output directories:**
- `./convert/buildconfigs/<namespace>/` — All extracted BuildConfigs
- `./convert/builds/<namespace>/` — All migrated Build resources and supporting files

You will see detailed logs for each BuildConfig. Here's an example output:

```
INFO[0000] Converting BuildConfigs in namespace: default 
INFO[0002] Found 1 BuildConfig to convert               
INFO[0002] -------------------------------------------------------- 
INFO[0002] Processing BuildConfig: ruby-source-git at index 0 
INFO[0002] -------------------------------------------------------- 
INFO[0002] Source strategy detected for BuildConfig: ruby-source-git 
INFO[0002] Processing From field: quay.io/prathore/ruby-27:latest 
INFO[0002] Registry PullSecret validated and service account "ruby-source-git" generated 
INFO[0002] Processing Environment Variables             
WARN[0002] Incremental build is not yet supported in the built-in Source-to-Image ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1607 
WARN[0002] ForcePull flag is not yet supported in the built-in Source-to-Image ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1606 
WARN[0002] Unlike BuildConfig, Volumes have to be supported in the Source-to-Image Strategy first in Shipwright. Please raise your requirements here: https://issues.redhat.com/browse/BUILD-1747 
INFO[0002] Processing Git Source                        
INFO[0002] Processing Git CloneSecret                   
WARN[0002] ConfigMaps are not yet supported in Shipwright build environment. RFE: https://issues.redhat.com/browse/BUILD-1745 
WARN[0002] Secrets are not yet supported in Shipwright build environment. RFE: https://issues.redhat.com/browse/BUILD-1744 
WARN[0002] Push to Openshift ImageStreams is not yet supported in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1756 
```

**Understanding the logs:**
- **INFO** — Successfully migrated fields
- **WARN** — Feature not yet supported but on the roadmap (includes RFE links)
- **ERRO** — Feature not planned for support

**Generated OpenShift Build file:**

The Build resource defines a ClusterBuildStrategy (source-to-image), Git source with URL and branch, environment variables and output registry. Also, please note that the from image in buildconfig is now passed as a "builder-image" parameter:

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  creationTimestamp: "2026-01-22T11:08:01Z"
  name: ruby-source-git
  namespace: default
spec:
  env:
  - name: RACK_ENV
    value: production
  output:
    image: image-registry.openshift-image-registry.svc:5000/default/ruby-source-git:latest
  paramValues:
  - name: builder-image
    value: quay.io/prathore/ruby-27:latest
  source:
    git:
      cloneSecret: github-ssh-secret
      revision: interim
      url: git@github.com:psrvere/ruby-hello-world.git
    type: Git
  strategy:
    kind: ClusterBuildStrategy
    name: source-to-image
status: {}
```

### Step 5: Run the Generated Build

**Review and modify if needed:**

First, check if there's a need to modify generated files. In our case, we may need to modify the output image. Although ImageStreams have been migrated to a correct URL in the generated file, this feature is not fully supported yet (see warnings).

You can change the output image to any public registry.

**Create a BuildRun resource** (`ruby_source_buildrun.yaml`):

```yaml
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  name: ruby-source-buildrun
  namespace: default
spec:
  build:
    name: ruby-source-git
  serviceAccount: ruby-source-git
```

**Run the BuildRun:**

```bash
oc apply -f ruby_source_buildrun.yaml
oc get buildrun
```