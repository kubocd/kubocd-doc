
# KuboCD

## Why KuboCD?

Most Kubernetes-ready applications provide a Helm chart. These charts are typically designed to be highly flexible and accommodate a wide range of use cases, which often makes their configuration complex.

Moreover, deploying an application using a Helm chart requires a deep understanding of the Kubernetes ecosystem. Consequently, application deployment often falls to platform administrators or platform engineers.

Even for experienced administrators, the verbosity of Helm configurations—especially the repetition of variables—can be tedious and error-prone. Industrializing these configurations is crucial for improving efficiency and reliability.

KuboCD is a tool that empowers Platform Engineers to package applications in a way that simplifies deployment for other technical users (such as Developers and AppOps) by abstracting much of the underlying infrastructure and environment complexity.

Beyond standard applications, KuboCD can provision core system components (e.g., ingress controllers, load balancers, Kubernetes operators), enabling the fully automated bootstrapping of a production-ready cluster from the ground up.

### When to Use KuboCD

KuboCD is particularly useful when:

- You want to standardize application deployment workflows across teams and environments without requiring every user to master Helm or Kubernetes internals.
- You are already using GitOps tools like Flux or ArgoCD and need a structured way to package and manage applications as versioned, portable artifacts.
- You want to encapsulate application configuration and logic into reusable, declarative units (Packages) that are decoupled from cluster-specific deployment scripts.
- You need to simplify developer access to existing Helm charts while enforcing consistency and best practices through curated Releases.
- You want to bootstrap entire environments—including base system components—in a fully automated manner.

## Main Concepts

KuboCD introduces two core concepts that form the foundation of its deployment model:

- **Package**:
  A Package is an OCI-compliant container image that bundles an application descriptor along with one or more Helm charts. It serves as a standardized deployment unit, encapsulating everything needed to describe and install an application.

- **Release**:
  A Release is a Kubernetes Custom Resource (CRD) that represents the deployment of a specific Package within a Kubernetes cluster. It defines how and where the application is deployed and manages its lifecycle.

## KuboCD, Flux, Helm, and GitOps

KuboCD is designed to integrate seamlessly with [Flux](https://fluxcd.io/){:target="_blank"}, enabling a fully automated GitOps workflow. While Flux handles continuous delivery (synchronizing changes from Git to the cluster), KuboCD simplifies application packaging and deployment logic, making the overall delivery pipeline more modular, maintainable, and user-friendly.

KuboCD is not a replacement for [Helm](https://helm.sh/){:target="_blank"}. Quite the opposite: it builds upon Helm’s proven capabilities and leverages the rich ecosystem of existing Helm charts.

Most production-grade applications provide an official or community-maintained Helm chart. KuboCD makes these charts more accessible by abstracting the complexity of Helm-based deployments.

By encapsulating Helm charts within standardized Packages and managing them via declarative Releases, KuboCD enables a broader audience—including those with less Helm expertise—to safely and efficiently deploy applications to Kubernetes.

### Feature Comparison: KuboCD vs. Helm vs. Flux

| Feature / Tool | KuboCD | Helm | Flux |
| :--- | :--- | :--- | :--- |
| **Primary Role** | Application packaging & deployment abstraction | Templating and deploying Kubernetes manifests | GitOps continuous delivery |
| **User Audience** | Platform Engineers, AppOps, Developers | DevOps, Kubernetes Experts | DevOps, SREs |
| **Ease of Use** | High (abstracts deployment logic) | Medium (requires Helm knowledge) | Medium (requires GitOps understanding) |
| **Supports GitOps** | ✅ (via integration with Flux) | ⚠️ (manual integration needed) | ✅ (native GitOps controller) |
| **Uses Helm Charts** | ✅ (packages & manages them) | ✅ (core functionality) | ✅ (can deploy HelmReleases) |
| **Custom Resources** | Release, Context | None (CLI and chart format) | HelmRelease, Kustomization, etc. |
| **Deployment Abstraction** | ✅ (encapsulates values, logic) | ❌ (user-defined values needed at deploy) | ❌ (relies on raw manifests or Helm) |
| **OCI Image Support** | ✅ (Packages are OCI images) | ✅ (since Helm v3.8+) | ✅ (via Helm OCI support) |
| **Ideal Use Case** | Standardizing deployments across teams | Managing complex app deployments manually | Automating deployments from Git |
