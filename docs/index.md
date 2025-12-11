

# KuboCD

## Why KuboCD ?

Most applications that can be deployed on Kubernetes come with a Helm chart. Moreover, this Helm chart is generally
highly flexible, designed to accommodate as many contexts as possible. This can make its configuration quite complex.

Furthermore, deploying an application on Kubernetes using an Helm Chart requires a deep understanding of the Kubernetes
ecosystem. As a result, application deployment is typically the responsibility of platform administrators or platform engineers.

And even for experienced administrators, the verbosity of Helm configurations, especially the repetition of variables,
can quickly become tedious and error-prone. Therefore, industrializing these configurations is crucial to improve
efficiency and reliability.

KuboCD is a tool that enables Platform Engineers to package applications in a way that simplifies deployment for other 
technical users (such as Developers, AppOps, etc.) by abstracting most of the underlying infrastructure and environment complexities.

In addition to usual applications, KuboCD can also provision core system components (e.g., ingress controllers, 
load balancers, Kubernetes operators, etc.), enabling fully automated bootstrapping of a production-ready cluster 
from the ground up.

### When to Use KuboCD

KuboCD is particularly useful when:

- You want to standardize application deployment workflows across teams and environments, without requiring everyone to master Helm or Kubernetes internals.
- You are already using GitOps tools like FluxCD or ArgoCD, and need a structured way to package and manage applications as versioned, portable artifacts.
- You want to encapsulate application configuration and logic into reusable, declarative units (Packages), decoupled from cluster-specific deployment scripts.
- You need to simplify access to existing Helm charts for developers, while enforcing consistency and best practices through curated Releases.
- You want to bootstrap entire environments (including base system components like ingress controllers, operators, etc.) in a fully automated way.

## Main concepts

KuboCD introduces two core concepts that form the foundation of its deployment model:

- **Package**:
  A Package is an OCI-compliant container image that bundles an application descriptor along with one or more Helm charts. 
  It serves as the standardized unit of deployment, encapsulating everything needed to describe and install an application.

- **Release**
  A Release is a custom Kubernetes resource that represents the deployment of a specific Package within a Kubernetes cluster. 
  It defines how and where the application is deployed, and manages the lifecycle of that deployment.

## KuboCD, Flux, Helm and GitOps

KuboCD is designed to seamlessly integrate with [Flux](https://fluxcd.io/){:target="_blank"}, enabling a fully automated GitOps workflow. 
While Flux handles the continuous delivery aspect (tracking changes in Git and applying them to the cluster), 
KuboCD simplifies application packaging and deployment logic, making the overall delivery pipeline more modular,
maintainable, and user-friendly.

KuboCD is not a replacement for [Helm](https://helm.sh/){:target="_blank"}. It is quite the opposite. It builds on top of Helm’s proven capabilities and 
leverages the rich ecosystem of existing Helm charts.

Most production-grade applications already provide an official or community-maintained Helm chart. 
KuboCD makes these charts more accessible by abstracting the complexity of Helm-based deployments.

By encapsulating Helm charts within standardized Packages and managing them via declarative Releases, 
KuboCD allows a broader audience (including less Helm-savvy users) to safely and efficiently deploy applications to Kubernetes.

### Feature Comparison: KuboCD vs Helm vs FluxCD

| Feature / Tool	| KuboCD           | Helm | FluxCD                       |
| --- |------------------| --- |------------------------------|
| Primary Role | Application packaging & deployment abstraction | Templating and deploying Kubernetes manifests | GitOps continuous delivery   |
| User Audience | Platform Engineers, AppOps, Developers | DevOps, Kubernetes Experts | DevOps, SREs | 
| Ease of Use | High (abstracts deployment logic) | Medium (requires Helm knowledge) | Medium (requires GitOps understanding) |
| Supports GitOps | ✅ (via integration with FluxCD) | ⚠️ (manual integration needed) | ✅ (native GitOps controller) |
| Uses Helm charts | ✅ (packages & manages them) | ✅ (core functionality) | ✅ (can deploy HelmReleases) |
| Custom Resources | Release, Context | None (CLI and chart format) | HelmRelease, Kustomization, etc. |
| Deployment Abstraction | ✅ (encapsulates values, logic) | ❌ (user-defined values needed at deploy) | ❌ (relies on raw manifests or Helm) |
| OCI Image Support | ✅ (Packages are OCI images) | ✅ (since Helm v3.8+) | ✅ (via Helm OCI support) |
| Ideal Use Case | Standardizing deployments across teams | Managing complex app deployments manually | Automating deployments from Git |


