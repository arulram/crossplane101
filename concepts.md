# Crossplane concepts overview

Think of Crossplane as a tool that upgrades Kubernetes from a container manager into a universal remote control for managing cloud infrastructure.

## Four core pieces

### 1. Crossplane (the remote control)

Normally, Kubernetes only cares about resources running inside your cluster, such as pods and containers. Crossplane extends Kubernetes so it can create, update, and manage resources outside the cluster, such as S3 buckets, databases, and DNS records.

It lets you manage cloud infrastructure using standard Kubernetes YAML files with `kubectl apply`.

### 2. Providers (the translators)

Crossplane does not know how to talk to every cloud provider by itself. A Provider is a plugin that teaches Crossplane the language of a specific service, such as `provider-aws`, `provider-gcp`, `provider-kafka`, or `provider-sql`.

Installing `provider-aws` tells Crossplane: "Here are all the AWS resource types that exist, and here is how to create them."

### 3. ProviderConfigs (credentials and settings)

Installing a Provider is not enough; it still needs to know where to log in. A ProviderConfig holds the authentication settings and target details for a specific cloud environment.

- It links the Provider to an AWS IAM role, service account key, or secret token.
- You can have multiple ProviderConfigs, for example one pointing to a dev AWS account and another pointing to prod.

### 4. Functions (the logic engine)

When building custom, reusable platform services in Crossplane, logic is often needed. For example, if the user passes `environment: prod`, create 3 database replicas; otherwise create 1.

Functions are small programs, written in Go, Python, or templating languages, that run inside the cluster to dynamically generate the exact infrastructure code needed.

## How they work together

```text
[ Your YAML ] --> [ Function ] --> [ Provider + ProviderConfig ] --> [ AWS / GCP / Azure ]
 (Request)        (Generates        (Translates & authenticates)       (Creates real resources)
                  custom logic)
```

## XRDs and Compositions

If Providers and Functions are the raw building blocks, XRDs and Compositions are how those raw materials become custom, easy-to-use services for developers.

Think of Compositions and XRDs as creating a custom cloud API.

### 5. XRD - Composite Resource Definition (the order form)

An XRD defines the interface. It tells Crossplane: "Create a brand new custom resource type, and here are the inputs users are allowed to customize."

Instead of forcing a developer to configure 50 complex cloud settings, an XRD can expose just a few simple fields:

- `dbSize`: `small` / `medium` / `large`
- `storageGB`: for example `20G`
- `environment`: `dev` / `prod`

Analogy: an XRD is like the menu at a restaurant. It lists what dishes are available and which options you can pick, but it does not contain the kitchen instructions.

### 6. Composition (the kitchen recipe)

A Composition is the actual blueprint behind the menu item. It tells Crossplane: "When someone fills out that order form, here is the exact set of real cloud resources to create under the hood, and how to map their simple inputs to those resources."

For example, when a user requests a `SimpleDatabase`, the Composition might create:

1. A PostgreSQL instance.
2. A security group with default ingress rules.
3. A subnet group across multiple availability zones.
4. A secret containing the generated credentials.

## How XRDs and Compositions work together

```text
[ Developer Submits Request ]
       |
       v  (Validates inputs against the menu)
   [  XRD  ]
       |
       v  (Looks up the recipe for those inputs)
[ Composition ]
       |
       +--> Creates PostgreSQL Instance
       +--> Creates Security Group
       +--> Creates DB Subnet Group
```

## Why use them

Without XRDs and Compositions, every developer who needs a database has to write hundreds of lines of complex YAML for networking, security, and storage.

With them:

1. The platform team writes the XRD and Composition once, encoding company security policies and best practices.
2. Developers request infrastructure using a clean 10-line YAML file without worrying about low-level cloud configuration.

## Quick terminology

- **XRD** => The schema / API definition.
- **Composition** => The infrastructure blueprint.
- **XR / Claim (XRC)** => The actual request created by a user.

## 7. Managed Resources (MRs) — the actual cloud objects

A Managed Resource (MR) is the direct Kubernetes representation of a single real-world cloud resource.

When Providers are installed, they add hundreds of these MR definitions into the cluster.

Examples include `RDSInstance`, `S3Bucket`, `GKECluster`, and `SecurityGroup`.

### Key trait: continuous reconciliation

Crossplane constantly watches MRs. If someone manually deletes an S3 bucket in the AWS Console, the Managed Resource controller detects the drift and recreates or fixes it.

In simple terms, MRs are the low-level atoms of cloud infrastructure in Kubernetes.

## 8. Packages — the app store / installers

A Package is Crossplane's mechanism for bundling and installing extensions into the control plane. Packages are distributed as OCI container images, similar to Docker images.

Crossplane has three main package types:

- **Provider packages**: Bundle Managed Resource definitions and controller code for a specific cloud, such as `provider-aws`.
- **Function packages**: Bundle reusable rendering or logic code, such as `function-patch-and-transform` or custom Go/Python functions.
- **Configuration packages**: Bundle custom XRDs and Compositions into a single versioned artifact that can be shipped across regions or clusters.

In simple terms, Packages are like installer files. They let you install Providers, Functions, or entire platform configurations with one command.

## 9. Operations — one-time or scheduled tasks

Most of Crossplane is built around continuous reconciliation, meaning it keeps a database or other resource present over time. Some tasks do not fit that model because they are one-time or scheduled actions that run to completion.

An Operation runs a pipeline of Functions once, like a Kubernetes Job, or on a schedule, like a CronJob, to perform operational maintenance.

Examples include rotating SSL certificates, running a database backup, validating security configurations, or triggering a rolling upgrade.

In simple terms, if Managed Resources declare what should exist, Operations run specific maintenance jobs.

## How everything fits together

```text
  +---------------------------------------------------------+
  |                 DEVELOPER / USER                        |
  |  Submits a simple YAML request (Claim / Composite CR)   |
  +----------------------------+----------------------------+
                               |
                               v
  +---------------------------------------------------------+
  |                     XRD (Menu)                          |
  |  Exposes the custom fields allowed for the request      |
  +----------------------------+----------------------------+
                               |
                               v
  +---------------------------------------------------------+
  |                 COMPOSITION (Blueprint)                 |
  | Uses Functions to render low-level Managed Resources    |
  +----------------------------+----------------------------+
                               |
                               v
  +---------------------------------------------------------+
  |              MANAGED RESOURCES (Cloud Atoms)            |
  |  Concrete objects: RDSInstance, S3Bucket, SecurityGroup |
  +----------------------------+----------------------------+
                               |
                               v
  +---------------------------------------------------------+
  |               PROVIDER & PROVIDER CONFIG                |
  | Authenticates and makes API calls to AWS / GCP / Azure  |
  +---------------------------------------------------------+
```

## Where Packages and Operations fit

Packages are the delivery trucks:

- Provider Packages install Managed Resources.
- Function Packages install Composition logic.
- Configuration Packages deploy XRDs and Compositions.

Operations run alongside this loop. While the main loop handles resource creation and continuous drift management, Operations handle day-2 maintenance tasks such as backups or certificate checks.
