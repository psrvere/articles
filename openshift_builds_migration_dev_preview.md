# Migrating BuildConfigs to OpenShift Builds Using Crane

**Use Crane to automatically convert your OpenShift BuildConfig resources to OpenShift Builds and embrace the next generation of container build infrastructure**

> 🚀 **Developer Preview**: This feature is now available for early adopters! Try it out and let us know what you think.

---

## Introduction

For years, OpenShift BuildConfig has been the go-to solution for building container images directly on OpenShift clusters. As the cloud-native ecosystem evolves, a new generation of build tooling has emerged.

[**OpenShift Builds**](https://github.com/redhat-openshift-builds/) is Red Hat's extensible build framework for building container images on OpenShift. It is based on [**Shipwright**](https://github.com/shipwright-io/), the upstream open-source project that provides a vendor-neutral framework for building container images on Kubernetes.

Built on top of Openshift Pipelines (Tekton), Openshift Builds offers a flexible, extensible approach to container builds with enterprise support from Red Hat.

Today, we're excited to announce a **Developer Preview** of BuildConfig to OpenShift Builds migration support **using [Crane](https://github.com/migtools/crane)**. This new upstream capability in Crane enables you to automatically convert your existing OpenShift BuildConfig resources to OpenShift Builds resources.

---

## What is Crane?

[Crane](https://github.com/migtools/crane) is an open-source migration tool under the [Konveyor](https://www.konveyor.io/) community that helps application owners migrate Kubernetes workloads and their state between clusters. It is designed on an Extract, Transform, and Apply framework.

With this new BuildConfig conversion capability, Crane extends its migration toolkit to help organizations modernize their container build infrastructure.

> ⚠️ **Developer Preview Notice**: This BuildConfig migration feature is currently available as an **upstream community capability** in Crane. It is not yet included in official Red Hat product offerings. As a Developer Preview, it is intended for evaluation and feedback purposes and is not supported for production use.

---

## What Gets Migrated?

Crane handles the heavy lifting of converting your BuildConfigs to OpenShift Builds. Here's what's supported:

### Build Strategies

| BuildConfig Strategy | OpenShift Builds ClusterBuildStrategy |
|:------------------------------:|:-------------------------------------:|
|        Docker Strategy         |               Buildah                 |
|        Source Strategy         |           Source to Image             |

### Source Types

The tool intelligently handles all major source types:

- **Git Sources**: Full support including repository URL, branch/tag references, and clone secrets
- **Binary Sources**: Converted to Local source type with streaming support
- **Image Sources**: Converted to OCI Artifact sources for image-based builds

### Key Features Migrated

✅ **Environment Variables** — Seamlessly transferred to Build spec  
✅ **Build Arguments** — Docker build args converted to ParamValues  
✅ **Dockerfile Paths** — Custom Dockerfile locations preserved  
✅ **Git Proxy Configuration** — HTTP/HTTPS proxy settings converted to environment variables  
✅ **Source Secrets** — Clone secrets for private Git repositories  
✅ **Push Secrets** — Registry authentication for image pushing  
✅ **Context Directories** — Build context paths maintained  

---

## How It Works

The migration process follows a following workflow:

1. **Discovery**: Crane connects to your cluster and lists all BuildConfigs in the specified namespace
2. **Analysis**: Each BuildConfig is analyzed for its strategy type, source configuration, and output settings
3. **Validation**: Pull secrets are validated to ensure they contain the required credentials
4. **Conversion**: Resources are mapped to their OpenShift Builds equivalents

---

## Getting Started

Ready to try the Developer Preview? Here's how to get started:

### Prerequisites

1. Openshift Pipelines installed on your cluster
2. OpenShift Builds installed on your cluster

---

## Example: Migration of BuildConfig with Docker Strategy and Git Source

Let's walk through a complete migration example step by step.

### Step 1: Create the BuildConfig

Let's take an example of the following BuildConfig (`ruby_docker_git_buildconfig.yaml`):

- **`spec.source`** is of Git type and defines a GitHub URI and branch. Source also defines context directory, inline dockerfile, secrets, and configMaps which are available in the builder pod during the build process.
- **`spec.strategy`** is of Docker type, defines a `from` image which overrides the builder image in the project's Dockerfile. It also defines a pull secret to pull images from a private registry, flags (`noCache`, `forcePull`, `imageOptimizationPolicy`), volumes, environment variables, and build args.

> **Note:** The `git.uri` field (`https://github.com/psrvere/ruby-hello-world`) points to a sample source code repository for this example. The `quay.io/prathore/ruby-27:latest` image is a private registry that you will not have access to. Please substitute your own source repository and registry credentials to try this example.

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ruby-docker-git
  namespace: default
spec:
  source:
    type: Git
    git:
      ref: interim
      uri: https://github.com/psrvere/ruby-hello-world
    contextDir: .
    dockerfile: |
      FROM ruby:2.7
      WORKDIR /app
      COPY . .
      RUN bundle install
      EXPOSE 4567
      CMD ["ruby", "app.rb"]
    secrets:
      - secret:
          name: database-credentials
        destinationDir: config
    configMaps:
      - configMap:
          name: app-config
        destinationDir: config
  strategy:
    type: Docker
    dockerStrategy:
      from:
        kind: DockerImage
        name: quay.io/prathore/ruby-27:latest
      pullSecret:
        name: registry-creds
      noCache: true
      env:
        - name: RACK_ENV
          value: production
      forcePull: true
      dockerfilePath: ./dockerfilelocation/Dockerfile
      buildArgs:
        - name: BUILD_VERSION
          value: 1.0.0
      imageOptimizationPolicy: SkipLayers
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

oc create imagestream ruby-docker-git
oc apply -f ruby_docker_git_buildconfig.yaml
oc start-build ruby-docker-git
oc get build
```

### Step 3: Download Crane CLI

Download the latest Crane CLI tool from [TO BE UPDATED](https://github.com/migtools/crane/releases/) for your platform.

<!-- TBD: Add curl commands after release and testing of crane binary. -->

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
INFO Converting BuildConfigs in namespace: default 
INFO Found 1 BuildConfig to convert              
INFO -------------------------------------------------------- 
INFO Processing BuildConfig: ruby-docker-git at index 0 
INFO -------------------------------------------------------- 
INFO Docker strategy detected                     
WARN From Field in BuildConfig's Docker strategy is not yet supported in built-in Buildah ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1746 
WARN NoCache flag is not yet supported in the built-in Buildah ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1578 
INFO Processing Environment Variables             
WARN ForcePull flag is not yet supported in the built-in Buildah ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1580 
INFO Processing Dockerfile path                   
INFO Processing Build Args                        
WARN ImageOptimizationPolicy (--squash) flag is not yet supported in the built-in Buildah ClusterBuildStrategy in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1581 
WARN Unlike BuildConfig, Volumes have to be supported in the Buildah Strategy first in Shipwright. Please raise your requirements here: https://issues.redhat.com/browse/BUILD-1747 
ERRO Inline Dockerfile is not supported in buildah strategy. Consider moving it to a separate file. 
INFO Processing Git Source                        
INFO Processing Git CloneSecret                   
WARN ConfigMaps are not yet supported in Shipwright build environment. RFE: https://issues.redhat.com/browse/BUILD-1745 
WARN Secrets are not yet supported in Shipwright build environment. RFE: https://issues.redhat.com/browse/BUILD-1744 
WARN Push to Openshift ImageStreams is not yet supported in Shipwright. RFE: https://issues.redhat.com/browse/BUILD-1756 
```

**Understanding the logs:**
- **INFO** — Successfully migrated fields
- **WARN** — Feature not yet supported but on the roadmap (includes RFE links)
- **ERRO** — Feature not planned for support

**Generated OpenShift Build file:**

The Build resource defines a ClusterBuildStrategy (Buildah), Git source with URL and branch, environment variables, build args, custom Dockerfile location, and output registry:

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  creationTimestamp: "2025-12-16T11:17:37Z"
  name: ruby-docker-git
  namespace: default
spec:
  env:
    - name: RACK_ENV
      value: production
  output:
    image: image-registry.openshift-image-registry.svc:5000/default/ruby-docker-git:latest
  paramValues:
    - name: dockerfile
      value: dockerfilelocation/Dockerfile
    - name: build-args
      values:
        - value: BUILD_VERSION=1.0.0
  source:
    git:
      revision: interim
      url: https://github.com/psrvere/ruby-hello-world
    type: Git
  strategy:
    kind: ClusterBuildStrategy
    name: buildah
```

### Step 5: Run the Generated Build

**Review and modify if needed:**

First, check if there's a need to modify generated files. In our case, we may need to modify the output image. Although ImageStreams have been migrated to a correct URL in the generated file, this feature is not fully supported yet (see warnings).

You can change output image to any public regitry.

**Create a BuildRun resource** (`ruby_docker_buildrun.yaml`):

```yaml
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  name: ruby-docker-buildrun
  namespace: default
spec:
  build:
    name: ruby-docker-git
  serviceAccount: ruby-docker-git
```

**Run the BuildRun:**

```bash
oc apply -f ruby_docker_buildrun.yaml
oc get buildrun
```

---

If you prefer video format, you can also view a video [demo](https://www.youtube.com/watch?v=6Z7brDRwF8s).

---

## Current Availability & Support

This BuildConfig migration capability is currently available as an **upstream Developer Preview** in the [Crane project](https://github.com/migtools/crane)

This feature is not yet included in official Red Hat product offerings. We encourage you to try the Developer Preview and share your feedback as we work toward broader availability.

For questions or issues, please [open an issue](https://github.com/migtools/crane/issues) on the Crane GitHub repository.

---

## Resources

- **Crane GitHub Repository**: [https://github.com/migtools/crane](https://github.com/migtools/crane)
- **Konveyor Community**: [https://www.konveyor.io/](https://www.konveyor.io/)
- **OpenShift Builds GitHub**: [https://github.com/redhat-openshift-builds/](https://github.com/redhat-openshift-builds/)
- **Shipwright (Upstream) GitHub**: [https://github.com/shipwright-io/](https://github.com/shipwright-io/)
- **Shipwright Documentation**: [https://shipwright.io/docs/](https://shipwright.io/docs/)
- **Crane Migration Demo**:: [https://www.youtube.com/watch?v=6Z7brDRwF8s](https://www.youtube.com/watch?v=6Z7brDRwF8s)

---