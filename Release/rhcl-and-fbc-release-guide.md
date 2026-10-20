# Red Hat Connectivity Link (RHCL) Release Guide

## Introduction

This guide explains how to release Red Hat Connectivity Link (RHCL) components and file-based catalogs using Konflux. It covers the complete release process from building container images to publishing operators in OpenShift OperatorHub.

The guide is designed for team members new to Konflux and includes:

- An overview of Konflux and RHCL's architecture
- Definitions of key terminology
- Step-by-step release procedures for both components and catalogs
- Troubleshooting guidance for common issues

---

## What is Konflux?

**Konflux** (formerly known as Red Hat Trusted Application Pipeline or RHTAP) is Red Hat's modern CI/CD platform for building, testing, and releasing containerized applications. It provides:

- **Automated Build Pipelines**: Triggered by git commits to build container images
- **Security Scanning**: Automated vulnerability scanning (Clair, Snyk, ClamAV, Coverity)
- **Enterprise Contract (EC) Validation**: Policy enforcement to ensure images meet Red Hat's release requirements
- **Release Orchestration**: Automated release pipelines that push images to registries and update catalogs

For RHCL, Konflux manages:
- Building container images for operators and supporting components
- Running security and compliance checks
- Releasing images to `registry.redhat.io` (production) and `registry.stage.redhat.io` (staging)
- Publishing file-based catalogs (FBCs) to OpenShift indexes

---

## Key Terminology

### Konflux-Specific Terms

| Term | Definition |
|------|------------|
| **Workspace (Tenant)** | An isolated environment in Konflux where applications are defined, built, and tested. RHCL uses the `api-management-tenant` workspace. |
| **Application** | A logical grouping of related components. RHCL has multiple applications (e.g., `rh-connectivity-link`, `rhcl-file-based-catalog`). |
| **Component** | A single container image built from a git repository. Examples: `rhcl-operator`, `dns-operator`, `authorino`. |
| **PipelineRun** | An execution of a Tekton pipeline. Each git push triggers a PipelineRun for building and testing. |
| **Snapshot** | A collection of component images at specific versions, representing a testable/releasable unit. |
| **Release** | The act of promoting a Snapshot to a managed namespace for distribution to customers. |
| **ReleasePlan** | A resource in the developer workspace declaring intent to release an application. |
| **ReleasePlanAdmission (RPA)** | A resource in the managed namespace (`rhtap-releng-tenant`) that defines how and where releases are executed. |

### Release-Specific Terms

| Term | Definition |
|------|------------|
| **Enterprise Contract (EC)** | A policy validation framework that checks built images against security and compliance rules before release. |
| **Enterprise Contract Policy (ECP)** | Specific policy rules that images must pass (e.g., no critical CVEs, signed builds, source containers included). |
| **File-Based Catalog (FBC)** | A declarative YAML format for defining OLM operator catalogs. RHCL uses FBC to publish operator metadata to OpenShift indexes. |
| **IIB (Index Image Builder)** | Red Hat's service for building and publishing operator index images. FBC fragments are submitted to IIB for processing. |
| **Bundle** | An OLM operator bundle containing operator manifests and metadata. Bundles are referenced in FBC fragments. |
| **Service Account** | Kubernetes service account with credentials for pushing to registries or indexes. Different accounts are used for staging vs. production. |
| **Pyxis** | Red Hat's container registry metadata service. Delivery repositories must be defined in Pyxis before releasing (see [Pyxis Repository Configs](#pyxis-repository-configs)). |
| **Cicada** | Red Hat tooling that creates and updates Pyxis delivery repositories from YAML files in the `pyxis-repo-configs` repository. |
| **Nudge** | An automatic pull request that updates a downstream component with the new image digest of an upstream one (e.g. operator image → bundle). See [Automation Overview](#automation-overview). |
| **Submodule** | Each `*-product-build` repo wraps an upstream Kuadrant repo as a git submodule. Bumping the submodule to a new tag is how a new upstream version enters the build. |
| **Constraints** | JSON Schema validation rules that restrict what values can be used in ReleasePlanAdmissions (e.g., allowed registries, policies). |
| **CPE** | A CPE (Common Platform Enumeration) is a Red Hat identifier assigned to a particular product including the product version. [More info](https://spaces.redhat.com/pages/viewpage.action?pageId=99849176&spaceKey=PRODSEC&title=CPE) The CPE example for RHCL is: `cpe:/a:redhat:connectivity_link:1::el9` |

---

## RHCL Application Architecture

RHCL uses a **version-based application structure** in Konflux, where each product version and operator has its own dedicated application. This provides clear separation between releases and allows independent versioning.

### Application Organization Pattern

Applications follow the naming convention: `rhcl-{version}-{operator-name}`

- **Version**: Product version (`1-4` or `1-5`)
- **Operator**: The specific operator (e.g., `authorino`, `dns-operator`, `limitador`, `rhcl-operator`)

### Applications in Konflux

1. rhcl-{version}-authorino-operator
2. rhcl-{version}-dns-operator
3. rhcl-{version}-limitador-operator
4. rhcl-{version}-rhcl-operator
5. rhcl-{version}-coredns (added in 1.2)
6. rhcl-{version}-mcp-gateway (added in 1.4)
7. rhcl-{version}-helm-charts (added in 1.5)
8. rhcl-{version}-rhcl-rhdh-dynamic-plugin (added in 1.5)
9. rhcl-file-based-catalog (shared by all versions)

> Applications 1–6 are created from the Konflux template (see [Where the Konflux Resources Are Defined](#where-the-konflux-resources-are-defined)). The helm-chart and dynamic-plugin applications have their own ReleasePlanAdmissions and are released separately from the operators.

---

### Component Reference for 1.4 and 1.5

Only the 1.4 and 1.5 streams are maintained. Names below use `{v}` for the hyphenated version (`1-4` or `1-5`). All components are built from the branch `rhcl-{version}` (`rhcl-1.4`, `rhcl-1.5`) of the repo shown. Operator bundle components (production, `-dev`, `-stage`) use the same repo and branch as their operator, with Containerfiles named `Containerfile.<name>-bundle`, `-bundle-dev`, `-bundle-stage`. Repos are in `https://github.com/rh-api-management/`.

#### Applications and components shared by 1.4 and 1.5

| Application | Component | Source repo | Containerfile |
|-------------|-----------|-------------|---------------|
| `rhcl-{v}-rhcl-operator` | `rhcl-{v}-rhcl-operator` (+ `-bundle`, `-bundle-dev`, `-bundle-stage`) | `rhcl-operator-product-build` | `Containerfile.rhcl-operator` |
| | `rhcl-{v}-rhcl-console-plugin` | `rhcl-console-plugin-product-build` | `Containerfile.console-plugin` |
| | `rhcl-{v}-rhcl-console-plugin-0-1-5` | `rhcl-console-plugin-product-build` | `Containerfile.console-plugin-0.1.5` |
| | `rhcl-{v}-wasm-shim` | `wasm-shim-product-build` | `Containerfile.wasm-shim` |
| | `rhcl-{v}-developer-portal-controller` | `developer-portal-controller-product-build` | `Containerfile.developer-portal-controller` |
| `rhcl-{v}-authorino-operator` | `rhcl-{v}-authorino` | `authorino-product-build` | `Containerfile.authorino` |
| | `rhcl-{v}-authorino-operator` (+ bundles) | `authorino-operator-product-build` | `Containerfile.authorino-operator` |
| `rhcl-{v}-dns-operator` | `rhcl-{v}-dns-operator` (+ bundles) | `dns-operator-product-build` | `Containerfile.dns-operator` |
| `rhcl-{v}-limitador-operator` | `rhcl-{v}-limitador` | `limitador-product-build` | `Containerfile.limitador` |
| | `rhcl-{v}-limitador-operator` (+ bundles) | `limitador-operator-product-build` | `Containerfile.limitador-operator` |
| `rhcl-{v}-coredns` | `rhcl-{v}-coredns` | `coredns-product-build` | `Containerfile.coredns` |
| `rhcl-{v}-mcp-gateway` | `rhcl-{v}-mcp-gateway` | `mcp-gateway-product-build` | `Containerfile.mcp-gateway` |
| | `rhcl-{v}-mcp-gateway-operator` (+ bundles) | `mcp-gateway-product-build` | `Containerfile.mcp-gateway-operator`; bundles use `Containerfile.bundle`, `.bundle-dev`, `.bundle-stage` |

#### Only in 1.5

| Application | Component | Source repo | Containerfile |
|-------------|-----------|-------------|---------------|
| `rhcl-1-5-rhcl-operator` | `rhcl-1-5-rhcl-console-plugin-0-6-0` | `rhcl-console-plugin-product-build` | `Containerfile.console-plugin-0.6.0` |
| `rhcl-1-5-helm-charts` | `rhcl-1-5-rhcl-operator-helm-chart` (+ `-dev`, `-stage`) | `rhcl-operator-product-build` | built from `helm-chart/`, `helm-chart-dev/`, `helm-chart-stage/` |
| `rhcl-1-5-rhcl-rhdh-dynamic-plugin` | `rhcl-1-5-rhcl-rhdh-dynamic-plugin` | `rhcl-rhdh-dynamic-plugin-product-build` | see repo `README`/`build.sh` |

> The helm-chart and dynamic-plugin applications are not part of the template. They have their own RPAs and ReleasePlans and are released separately.

#### Build nudging differs between the streams

| | 1.4 (`rhcl-template`) | 1.5 (`rhcl-template-olm-v1`) |
|---|---|---|
| Operand/operator builds (authorino, limitador, dns, coredns, mcp-gateway) nudge | their own `...-bundle` | **`rhcl-{v}-rhcl-operator`** |
| rhcl-operator, console plugins, wasm-shim, developer-portal-controller nudge | `rhcl-{v}-rhcl-operator-bundle` | `rhcl-{v}-rhcl-operator-bundle` |

The 1.5 template still builds per-component bundles, but they are not delivered to customers or added to the catalog.

---

#### 1.4 vs 1.5: how the operators are delivered

RHCL 1.4 and 1.5 are the only streams currently maintained. They differ in **how the component operators reach a cluster**:

| | **1.4** | **1.5** |
|---|---------|---------|
| Install model | Each operator (authorino, dns, limitador, MCP Gateway) ships its **own bundle** and is installed by OLM as a **dependency** of the RHCL operator | The **RHCL operator deploys everything**. Component manifests are generated as component charts inside the RHCL operator build |
| Template | `rhcl-template` | `rhcl-template-olm-v1` |
| OLM channel | `stable-1.4` | `stable-1.5` (set in `bundle-generation/rhcl-operator.yaml`) |
| Where component image digests go | Nudge each component's own bundle | Nudge `rhcl-operator`; `Generate Component Charts` / `Generate Helm Chart` regenerate the charts |
| Bundles to verify in a Snapshot | One per operator | The RHCL operator bundle is the one that matters |
| FBC packages | Add each operator's bundle to its own package | **Only the `rhcl-operator` package.** Component operators have no catalog entries in 1.5 |
| FBC channel | `stable-1.4` (`stable` is the old channel; do not add new versions to it) | `stable-1.5` |
| Extra artifacts | None | Helm charts and RHDH dynamic plugin releases |

**What this means when releasing**
- For 1.4, follow the guide as written: verify and release every operator's bundle, then add each bundle to the catalog.
- For 1.5, the operand images (authorino, limitador, dns-operator, ...) are still built and released as images, but the **RHCL operator bundle** is what ties them together. Check that its CSV `relatedImages` and component charts point at the digests in your Snapshot (see [Step 1](#step-1-verify-builds-are-complete)).
- Per-component bundle components and RPAs for 1.5 still exist in `konflux-release-data`, but only the RHCL operator bundle is added to the catalog.

Branches use the dotted form (`rhcl-1.4`, `rhcl-1.5`); Tekton files are named `rhcl-1-4-*` / `rhcl-1-5-*`.

---

### Maintained Streams

**RHCL 1.5 is the last minor version.** There will be no 1.6, so there is no new-version setup to do (no new applications, branches, templates, or RPA sets). Only two streams are maintained, and all releases from here on are **patch releases** of one of them:

| Stream | Branch | Template | Catalog channel |
|--------|--------|----------|-----------------|
| 1.4 | `rhcl-1.4` | `rhcl-template` | `stable-1.4` |
| 1.5 | `rhcl-1.5` | `rhcl-template-olm-v1` | `stable-1.5` |

See [Patch Releases](#patch-releases) for the procedures. When 1.4 reaches end of life, its resources are simply retired; nothing replaces it.

#### Where the Konflux Resources Are Defined

Applications and components are rendered by the [Konflux template system](https://konflux-ci.dev/docs/patterns/managing-multiple-versions/). Each stream is a `ProjectDevelopmentStream` (e.g. `connectivity-link-1.5`, project `connectivity-link`) that points at a template. You should rarely need to touch them.

The source of truth is the GitOps config in [`konflux-release-data`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/tree/main/tenants-config/cluster/stone-prd-rh01/tenants/api-management-tenant/rhcl-project-streams) (`tenants-config/.../api-management-tenant/rhcl-project-streams/`). The older [rhcl-konflux-resources](https://github.com/rh-api-management/rhcl-konflux-resources) repo holds early copies of the templates.

**To change a component for an existing stream** (e.g. add a build or fix a Containerfile path), edit the template in `rhcl-project-streams/` and open an MR against `konflux-release-data`; it is applied to the cluster after merge. Then check the result:
```bash
oc get components -n api-management-tenant | grep rhcl-1-5
```
Template changes apply to every stream using that template, so make sure a change for 1.5 doesn't alter 1.4 (they use different templates).

---

#### File-Based Catalog Application

##### 9. **rhcl-file-based-catalog**
Contains FBC components for publishing operator metadata to OpenShift OperatorHub indexes.

**Components (one per OCP version):**
1. `rhcl-file-based-catalog-4-14` - OCP 4.14 catalog
2. `rhcl-file-based-catalog-4-15` - OCP 4.15 catalog
3. `rhcl-file-based-catalog-4-16` - OCP 4.16 catalog
4. `rhcl-file-based-catalog-4-17` - OCP 4.17 catalog
5. `rhcl-file-based-catalog-4-18` - OCP 4.18 catalog
6. `rhcl-file-based-catalog-4-19` - OCP 4.19 catalog
7. `rhcl-file-based-catalog-4-20` - OCP 4.20 catalog
8. `rhcl-file-based-catalog-4-21` - OCP 4.21 catalog
9. `rhcl-file-based-catalog-4-22` - OCP 4.22 catalog

**Source:** `https://github.com/rh-api-management/rhcl-file-based-catalog`
**Containerfiles:** `{version}.Containerfile` (e.g., `4.21.Containerfile`)

---

### Build Nudging

Components use **build-nudges-ref** to trigger dependent rebuilds:

**Example:** When `rhcl-1-4-rhcl-operator` is built, it automatically triggers builds for:
- `rhcl-1-5-rhcl-operator-bundle`
- `rhcl-1-5-rhcl-operator-bundle-stage`
- `rhcl-1-5-rhcl-operator-bundle-dev`

This ensures bundles always reference the latest operator images.

When a nudging build finishes, Konflux opens a pull request (branch `konflux/component-updates/...`) that writes the new image digest into the bundle inputs (`image-pullspecs`). Renovate settings in each repo auto-merge these PRs, so no human action is normally needed. See [Automation Overview](#automation-overview).

---

## Automation Overview

Much of the work between "upstream cut a release" and "images are ready to ship" is automated by GitHub Actions and bots in the `rh-api-management` repos. You don't need to run these by hand in the normal flow, but you do need to know they exist so you can tell whether a PR you see was machine-made and what to do when one stalls.

### The normal flow, end to end

```
Upstream Kuadrant tag (e.g. v1.5.0)
   │  daily "Submodule Version Bump" workflow
   ▼
PR: bump submodule + version label in *-product-build  (auto/bump-... branch)
   │  merge → Konflux builds the image
   ▼
Konflux nudge PR into the bundle / rhcl-operator repos  (image digests)
   │  auto-merged
   ▼
"Generate Bundles" / "Generate Helm Chart" workflows commit regenerated manifests
   │  merge → bundle + operator rebuild → Snapshot
   ▼
YOU: verify snapshot → RPA/ReleasePlan → Release (stage, then prod)
   ▼
YOU: add bundles to FBC → release FBC
```

### What runs where

| Automation | Where | Trigger | What it does |
|-----------|-------|---------|--------------|
| **Submodule Version Bump** | every `*-product-build` repo that wraps an upstream (authorino, authorino-operator, dns-operator, limitador, limitador-operator, wasm-shim, coredns, developer-portal-controller, console plugin, mcp-gateway, rhdh plugin) | Daily cron + manual | Finds the newest upstream semver tag, moves the submodule to it, updates the version label in the Containerfile, and opens a PR (`auto/bump-<name>-<tag>`). |
| **Generate Bundles** | `authorino-operator`, `dns-operator`, `limitador-operator`, `mcp-gateway`, `rhcl-operator` | PR/push that changes `image-pullspecs` or the operator's `*.yaml` config | Regenerates the `bundle/`, `bundle-dev/`, `bundle-stage/` manifests and commits them to the PR. Commit message: "Generate Bundles after change to image-pullspecs...". |
| **Generate Helm Chart / Component Charts** | `rhcl-operator-product-build` | PR/push changing upstream charts, `image-pullspecs`, or `rhcl-operator.yaml` | Regenerates `helm-chart`, `helm-chart-dev`, `helm-chart-stage` and `component-charts`. |
| **Konflux nudge PRs + auto-merge** | all repos with bundles | Konflux build completes | Opens `konflux/component-updates/...` PRs with new digests; merged automatically (label `konflux-nudge` in dns-operator, Renovate automerge elsewhere). |
| **MintMaker (Renovate)** | all repos | Weekly-ish | Opens PRs to refresh Konflux task references, base-image digests, and RPM lockfiles (`rpms.lock.yaml`). Security refreshes are labelled `[SECURITY]`. |
| **Update Pipeline Task References** | `rhcl-operator-product-build` | Manual | Runs `konflux-pipeline-patcher` to bump Tekton task bundles for one or all pipelines and opens a PR. Use when a task bundle digest in a pipeline is broken (several "Fix invalid rpms-signature-scan digest" fixes were this). |
| **Packaging checks** | `rhcl-rhdh-dynamic-plugin-product-build` | PR/push | Verifies `packaging/yarn.lock` matches the plugin and keeps `packaging/` in sync on submodule bumps. |
| **Add bundle skill** | `rhcl-file-based-catalog` | Manual (Claude Code `/add-bundle`) | Renders a bundle with `opm`, adds the channel entry, and validates catalogs. This is the recommended way to add FBC content. See [Step 1: Update FBC Catalog YAML](#step-1-update-fbc-catalog-yaml). |

### What is still manual

- Bumping the submodules to the upstream tags for this release when the daily bump hasn't already done it
- Reviewing and merging the daily bump PRs
- Fixing a build when a nudge PR or generated-manifest commit fails
- Creating the Release, and all FBC changes

---

## Understanding the Release Process

The RHCL release process follows this high-level flow:

### 1. **Build Phase** (Developer Workspace: `api-management-tenant`)
- Developer pushes code to GitHub (`rh-api-management` repository)
- Konflux triggers PipelineRuns based on `.tekton/*.yaml` files
- Each component is built using its Containerfile
- Dependencies are prefetched using Cachi2 (Go modules, RPMs, pip packages)
- Images are built hermetically (network-isolated for reproducibility)
- Built images are pushed to `quay.io/redhat-user-workloads/api-management-tenant/` and can be accessed from `image-rbac-proxy.apps.stone-prd-rh01.pg1f.p1.openshiftapps.com`
  - Registry login information can be found in `Component Details` page of the Konflux web application.

### 2. **Test & Scan Phase**
- Enterprise Contract validation runs against built images
- Security scans execute:
  - Clair (CVE scanning)
  - Snyk (dependency analysis)
  - ClamAV (malware detection)
  - Coverity (static analysis)
  - Shell check, unicode check
- Preflight checks verify Red Hat certification requirements
- A Snapshot is created containing all tested component versions

### 3. **Release Decision**
- If ReleasePlan has `auto-release: true`, a Release is created automatically
- If `auto-release: false`, team manually creates a Release resource
- The Release references a Snapshot and a ReleasePlan

### 4. **Release Phase** (Managed Workspace: `rhtap-releng-tenant`)
- Release Service matches the ReleasePlan to a ReleasePlanAdmission (RPA)
- The RPA defines:
  - Target registries (e.g., `registry.redhat.io`)
  - Image tags
  - Enterprise Contract Policy (stricter for production)
  - Release pipeline to execute
  - Service account with credentials
- Release pipeline executes:
  - Re-validates against production EC policy
  - Signs images with Red Hat production keys
  - Pushes to `registry.redhat.io` (production) or `registry.stage.redhat.io` (staging)
  - Updates Pyxis metadata
  - For FBC: submits to IIB and publishes to operator indexes

### 5. **Post-Release**
- Customers can pull images from `registry.redhat.io`
- Operators appear in OpenShift OperatorHub catalogs
- Release artifacts are archived for compliance

---

## Prerequisites

Before releasing RHCL, ensure you have:

### Access & Permissions
1. **Konflux Access**
   - Request access: https://konflux.pages.redhat.com/docs/users/getting-started/getting-access.html
   - Workspace: `api-management-tenant`
   - Managed workspace view access: `rhtap-releng-tenant`

2. **OpenShift CLI (`oc`)**
   - Install: https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html
   - Login to Konflux cluster

3. **GitLab Access**
   - Access to [`releng/konflux-release-data`](https://gitlab.cee.redhat.com/releng/konflux-release-data) repository

### Configuration Files
4. **ReleasePlanAdmissions (RPAs)**
   - Must be created per release basis.
   - Located in [`releng/konflux-release-data/config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/tree/main/config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management)
   - Files follow naming pattern: `rhcl-{version}-{component}-{environment}-rhel9.yaml` (`{environment}` is `stage` or `prod`). Helm charts use `rhcl-{version}-helm-charts-{environment}.yaml`.
   - RPAs are per component for 1.3+ (`authorino-operator`, `dns-operator`, `limitador-operator`, `rhcl-operator`, `coredns`, `mcp-gateway`). Older 1.1/1.2 files use `authorino`/`limitador` and, for 1.1, `staging`.
   - Image tags: RPAs use `{{labels.version}}` so the version tag follows the image's own label. The product tags (`rhcl-1.5`, `rhcl-1.5.0`) are edited by hand each release.
   - Releases can be blocked by setting the label `release.appstudio.openshift.io/block-releases: "true"` (the releng team does this during outages; check it is `"false"` if a release is refused).
   - Examples:
     - [`rhcl-1-5-rhcl-operator-prod-rhel9.yaml`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-1-5-rhcl-operator-prod-rhel9.yaml) (production)
     - [`rhcl-1-4-authorino-operator-stage-rhel9.yaml`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-1-4-authorino-operator-stage-rhel9.yaml) (staging)
     - [`rhcl-file-based-catalog-stage.yaml`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-file-based-catalog-stage.yaml) (FBC staging)
   - Example MR for creating new RPAs: [MR !10874](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/10874). For 1.5: [!22271](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/22271) (RPAs and ReleasePlans), [!23156](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/23156) (helm charts), [!23514](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/23514) (dynamic plugin)

5. **ReleasePlans (GitOps)**
   - ReleasePlans are no longer created by hand on the cluster. They live in [`konflux-release-data`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/tree/main/tenants-config/cluster/stone-prd-rh01/tenants/api-management-tenant/rhcl-releaseplans) under `tenants-config/cluster/stone-prd-rh01/tenants/api-management-tenant/rhcl-releaseplans/rhcl-{version}/` (one file per application, with a stage and a prod plan) and are applied by ArgoCD after merge.
   - All have `auto-release: "false"`, so every release is created manually.
   - Release notes text (description, synopsis, topic) currently lives here, but it can also be set in the RPA or the Release. Edit it only if the advisory text must change.

6. **Enterprise Contract Policies**

   RHCL uses these Enterprise Contract policies (each has a `-stage` twin for staging):
   - [`registry-api-management-rhcl-1-5-prod`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/EnterpriseContractPolicy/registry-api-management-rhcl-1-5-prod.yaml) - RHCL 1.5 operator RPAs only. It allows `NetworkPolicy` resources in bundles, which is safe only because 1.5 does not ship to OCP older than 4.20. Do not reuse it for other applications.
   - [`registry-api-management-chart-prod`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/EnterpriseContractPolicy/registry-api-management-chart-prod.yaml) - Helm chart releases
   - [`registry-api-management-prod`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/stone-prd-rh01.pg1f.p1/product/EnterpriseContractPolicy/registry-api-management-prod.yaml) - For component releases of 1.4 and earlier and for other 1.5 components (operators, bundles, images)
   - [`fbc-standard`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/config/common/product/EnterpriseContractPolicy/fbc-standard.yaml) - For file-based catalog releases

7. **Constraints**
   - Any new policy name must be added to the allowed list here, or `tox` fails
   - Located in: [`releng/konflux-release-data/constraints/product/api-management.yaml`](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/constraints/product/api-management.yaml)
   - Validates RPA values (registries, tags, policies)

8. **Pyxis delivery repositories** (only when adding a new image)
   - See [Pyxis Repository Configs](#pyxis-repository-configs). Existing images already have repositories, so a normal version or patch release needs nothing here.

9. **A Jira issue to track productization tasks**
   - Must be created per release (every release is now a patch release of 1.4 or 1.5)
   - Triggers a "Test Plan" Jira issue for the same release.
   - Has the following subtasks:
     - "Update releasePlanAdmissions (and ReleasePlans if the advisory text changes) for RHCL X.Y.Z"
     - "Prepare RC1": Created after the upstream release is available. Create other "Prepare RC N" subtasks as needed when release candidates have bugs.
     - "Ship Images and Bundles"
     - "Ship FBC"
   - Leave evidence of the work being done in subtask comments such as links to PRs, MRs, or relevant links.
   - Example issue: [JIRA 704](https://issues.redhat.com/browse/CONNLINK-704)

---

## Pyxis Repository Configs

Before an image can be released to `registry.redhat.io/<namespace>/<name>`, its **delivery repository** must exist in Pyxis. These are defined as code in [`pyxis-repo-configs`](https://gitlab.cee.redhat.com/releng/pyxis-repo-configs) and created by Cicada. You only touch this repo when adding a **new image** (a new component, or a repository moving between Tech Preview and GA). Version and patch releases of existing images need nothing here.

### Where RHCL repositories are defined

RHCL files are under `products/rhcl/`, one per operator group:

| File | Repositories (`rhcl-1/` unless noted) |
|------|------|
| `rhcl-operator.yaml` | `rhcl-rhel9-operator`, `rhcl-operator-bundle`, `wasm-shim-rhel9`, `rhcl-console-plugin-rhel9`, `developer-portal-controller-rhel9` |
| `authorino-operator.yaml` | `authorino-rhel9`, `authorino-rhel9-operator`, `authorino-operator-bundle` |
| `dns-operator.yaml` | `coredns-rhel9`, `dns-rhel9-operator`, `dns-operator-bundle` |
| `limitador-operator.yaml` | `limitador-rhel9`, `limitador-rhel9-operator`, `limitador-operator-bundle` |
| `mcp-gateway.yaml` | `mcp-gateway-rhel9`, `mcp-gateway-rhel9-operator` (GA, `rhcl-1/`); same names plus the operator bundle under `rhcl-tech-preview/` |
| `dynamic-plugin.yaml` | `rhcl-rhdh-dynamic-plugin-rhel9` in `rhcl-1/` (GA) and `rhcl-tech-preview/` |

Recent changes: Tech Preview repos for MCP Gateway (April 2026), the dynamic plugin tech-preview repo (July 2026), GA `rhcl-1/` repos for MCP Gateway (September 2026) and the dynamic plugin (October 2026).

### Adding a repository

1. Get `Developer` access to the repo (ask `@cicada-support` in #forum-cicada).
2. Create a branch **in the origin repo, not a fork**, and add an entry to the right file in `products/rhcl/`. Copy an existing entry and change:
   - `repository`: `namespace/name` (this must match the `url` in your RPA)
   - `image_type`: `Layered` (operand images), `Operator Controller` (operator images), or `Operator Bundle Image` (bundles)
   - `release_categories`: `Generally Available` or `Tech Preview`
   - `display_data` and `contacts`
3. Open a merge request. Product definitions go in their own MR, separate from product-listing changes. Reviewers from the RHCL CODEOWNERS list (`/products/rhcl/`) must approve.
4. After merge, the pipeline creates the repository in **both** production and stage Pyxis. Check that both jobs succeeded. Use `publish_envs` to publish to only one.
5. Then add the matching RPA mapping in `konflux-release-data` and release.

If a release fails at the Pyxis step with a "repository not found" type error, the repository is missing or its merge pipeline failed.

---

## Step-by-Step: Releasing RHCL Components

This section covers releasing operator images and bundles (not FBC).

### Step 1: Verify Builds are Complete

1. **Check PipelineRuns**
   ```bash
   oc project api-management-tenant
   oc get pipelineruns --sort-by=.metadata.creationTimestamp | tail -20
   ```

2. **Find Recent Successful Builds**
   - Look for PipelineRuns with `Succeeded` status
   - Note the Snapshot name (format: `{application}-{hash}`)

3. **Inspect Snapshot**
   ```bash
   oc get snapshot {snapshot-name} -o yaml
   ```
   - Verify all expected components are present
   - Note the component image digests

4. **Verify Bundle References**

   Bundle images contain a ClusterServiceVersion (CSV) that must reference the correct operand images from the same snapshot.

   **Extract and inspect bundle manifests:**
   ```bash
   SNAPSHOT="your-snapshot-name"
   BUNDLE_IMAGE=$(kubectl get snapshot $SNAPSHOT -o yaml | yq e '.spec.components[] | select(.name | test("-operator-bundle$")).containerImage' | head -1)
   echo "Bundle image: $BUNDLE_IMAGE"

   # Extract bundle manifests to local directory
   mkdir -p ./bundle-check
   oc image extract --confirm $BUNDLE_IMAGE --path /manifests/:./bundle-check

   # Use yq to extract relatedImages
   yq e '.spec.relatedImages' ./bundle-check/*.clusterserviceversion.yaml
   ```

   **What to verify:**
   - The bundle's CSV must reference operand images that are part of the same snapshot
   - For example, `rhcl-1-4-limitador-operator-bundle` must reference the `rhcl-1-4-limitador` image digest from the snapshot
   - All image references should use digests (sha256:...), not tags
   - Image digests in the CSV should match the digests in the snapshot

   **Compare with snapshot:**
   ```bash
   # Get operand image from snapshot (for limitador example)
   # Note: This matches the limitador image, NOT the limitador-operator
   OPERAND_IMAGE=$(kubectl get snapshot $SNAPSHOT -o yaml | yq e '.spec.components[] | select(.name | test("limitador$") and (.name | test("operator") | not)).containerImage')
   echo "Operand image in snapshot: $OPERAND_IMAGE"

   # Check if this digest appears in the bundle CSV
   DIGEST=$(echo $OPERAND_IMAGE | cut -d'@' -f2)
   grep $DIGEST ./bundle-check/*.clusterserviceversion.yaml
   ```

   **Common issue**: Auto-generated snapshots may include bundles that reference outdated or unreleased operand images

   **If references are incorrect:**
   - Manually create a new snapshot including the correct component versions
   - Trigger a rebuild of the bundle component to pick up the latest nudged references

### Step 2: Choose Staging or Production

- **Staging Release**: Test the release process without affecting customers
  - Target: `registry.stage.redhat.io`
  - RPA files end with: `-stage-rhel9.yaml`
  - Example: `rhcl-1-5-rhcl-operator-stage-rhel9.yaml`

- **Production Release**: Customer-facing release
  - Target: `registry.redhat.io`
  - RPA files end with: `-prod-rhel9.yaml`
  - Example: `rhcl-1-5-rhcl-operator-prod-rhel9.yaml`

### Step 3: Update or Create ReleasePlanAdmission

**Important**: RHCL has no new minor versions, so you will not create new RPAs. Each patch release reuses the stream's existing RPAs and only updates their tags.

For each component with updated versions:

1. **Clone the Release Data Repo**
   ```bash
   git clone git@gitlab.cee.redhat.com:releng/konflux-release-data.git
   cd konflux-release-data
   git checkout -b update-rhcl-1.4.4
   ```

2. **Edit the RPA File**
   ```bash
   vim config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-1-5-rhcl-operator-prod-rhel9.yaml
   ```

3. **Update Component Versions**
   - Locate the `mapping.components` section
   - Update `tags` for each component
   - Example (1.5 style, where the version tags follow the image label):
     ```yaml
     - name: rhcl-1-5-rhcl-operator
       repositories:
         - url: "registry.redhat.io/rhcl-1/rhcl-rhel9-operator"
           tags:
             - "v{{labels.version}}-{{digest_sha}}"
             - "{{labels.version}}"
             - "rhcl-1.5"
             - "rhcl-1.5.1"  # edit this product version tag each release
     ```
   - In older RPAs the tags are literal (e.g. `"0.2.1"`); update those by hand.

4. **Test Locally**
   ```bash
   tox
   ```
   - This runs validation tests, including constraint checks
   - Fix any errors before committing

5. **Commit and Push**
   ```bash
   git add config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/
   git commit -m "Update RHCL 1.4.4 production RPA"
   git push origin update-rhcl-1.4.4
   ```

6. **Create Merge Request**


7. **Merge the MR**
   - Post-merge pipeline will apply the RPA to `rhtap-releng-tenant`

### Step 4: Verify RPA is Applied

```bash
oc project rhtap-releng-tenant
oc get releaseplanadmission rhcl-1-5-rhcl-operator-prod-rhel9 -o yaml
```

- Verify the RPA matches your changes
- Check `status.conditions` for any errors

### Step 5: Create a ReleasePlan

**Important**: Patch releases re-use the existing ReleasePlans. You only change them if the release advisory text needs to change.

ReleasePlans are managed in git (see [Prerequisites](#prerequisites)), not applied by hand:

1. **Check whether the plans already exist**
   ```bash
   oc get releaseplan -n api-management-tenant | grep rhcl-1-5
   ```

2. **Only if the advisory text must change** and it is set in the ReleasePlan, edit the plan in `konflux-release-data` under `tenants-config/.../api-management-tenant/rhcl-releaseplans/rhcl-1-4/` or `rhcl-1-5/` (`spec.data.releaseNotes`: description, synopsis, topic). Merge the MR; ArgoCD applies it.

3. **Verify ReleasePlan is Matched**
   ```bash
   oc get releaseplan <your-releaseplan-name> -o yaml
   ```
   Check that `status.conditions` shows `Matched: True`

### Step 6: Create a Release

1. **Get an Existing Release as Template or use embedded example**
   ```bash
   # List recent releases to find an example
   oc get releases --sort-by=.metadata.creationTimestamp 

   # Use a specific release as template (replace with actual release name)
   oc get release rhcl-1-4-rhcl-operator-prod-20261015-123456 -o yaml > new-release.yaml
   ```
   

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: rhcl-1-4-4-rhcl-operator-prod-rhel9
  namespace: api-management-tenant
spec:
  data:
    releaseNotes:
      issues:
        fixed:
          - id: CONNLINK-529
            source: issues.redhat.com
          - id: CONNLINK-433
            source: issues.redhat.com
      type: RHBA
  gracePeriodDays: 7
  releasePlan: rhcl-1-4-rhcl-operator-prod
  snapshot: rhcl-1-4-rhcl-operator-rc2
```

2. **Set the advisory type (bug fix vs security)**

   The Release decides what kind of advisory is published, in `spec.data.releaseNotes`:

   | Release contains | `type` | Extra fields |
   |------------------|--------|--------------|
   | Only bug fixes | `RHBA` | `issues.fixed` (Jira issues) |
   | **Any security fix (CVE)** | **`RHSA`** | `issues.fixed` **and** a `cves` list |
   | Only enhancements (rare for patches) | `RHEA` | `issues.fixed` |

   If a release includes **any** security fix, it must be `RHSA`. Do not ship a CVE fix as `RHBA`.

   Security release example:
   ```yaml
   spec:
     data:
       releaseNotes:
         type: RHSA
         issues:
           fixed:
             - id: CONNLINK-1566
               source: issues.redhat.com
         cves:
           - key: CVE-2026-12345
             component: rhcl-1-4-limitador
           - key: CVE-2026-12345
             component: rhcl-1-4-wasm-shim
   ```
   - `key` is the CVE identifier.
   - `component` is the **Konflux component name** (as in the Snapshot) that contains the fix. List a CVE once per affected component.
   - Keep the Jira issues in `issues.fixed` as for any release.
   - **Advisory text is plain text that you write.** The synopsis, topic and description (`releaseNotes`) are not generated, and where they live (RPA, ReleasePlan or Release) depends on configuration. In RHCL today they are in the ReleasePlans under `rhcl-releaseplans/`. For a security release, word them as a security update and include the severity, as in the real example below: `Important: Red Hat Connectivity Link component - rhcl-operator security update`.
   - **Severity and CVE identifier come from the Red Hat CVE database.** Look up each CVE at `https://access.redhat.com/security/cve/<CVE-ID>` to get its exact identifier and its Red Hat severity rating (Low, Moderate, Important or Critical). When a release has several CVEs, use the highest severity in the synopsis.
   - `issues.fixed` can be left out if the release fixes only CVEs and no Jira issues.

   **Real example:** [RHSA-2026:83531](https://access.redhat.com/errata/RHSA-2026:83531) (RHCL 1.4, issued 2026-10-15). It was released from the `rhcl-1-4-rhcl-operator` application, which is why the advisory covers `wasm-shim`, `rhcl-console-plugin`, `developer-portal-controller`, `rhcl-operator` and the `rhcl-operator` bundle in one advisory. It lists one CVE, is rated Important, and has no Jira issues in its Fixes section. One advisory is published per application release, so a security patch that touches several applications produces several advisories.

   **Where to put these fields:** `issues` and `cves` belong **only on the Release**. Do not add them to the ReleasePlan or the RPA. The `konflux-release-data` tests reject them there.

   **Finding the CVEs:** collect them from the Jira issues for this release, then look each one up in the Red Hat CVE database (above) to confirm the identifier and severity. A security patch usually has several CVEs, so list all of them.

   Use the same `type` and `cves` on the stage Release as on the prod one, so staging exercises the same advisory path.

3. **Modify the Release**
   - Update `metadata.name` with unique timestamp
   - Update `spec.releasePlan` to your ReleasePlan name
   - Update `spec.snapshot` to your Snapshot name
   - Set `type` and, for RHSA, the `cves` list (previous step)
   - Update `issues.fixed` to the Jira issues in this release (replace the ones copied from the template)
   - Remove `metadata.creationTimestamp`, `metadata.generation`, `metadata.resourceVersion`, `metadata.uid`, and entire `status` section

4. **Apply the Release**
   ```bash
   oc apply -f new-release.yaml
   ```

5. **Monitor Release Pipeline**
   ```bash
   oc get release <your-release-name> -w
   ```

   Or view in the UI:
   - https://console.redhat.com/preview/application-pipeline/workspaces/api-management-tenant/releases

### Step 7: Monitor Release Execution

1. **Check Release Status**
   ```bash
   oc get release {release-name} -o yaml
   ```
   - Look at `status.conditions`
   - Note the `processing` and `succeeded` conditions

2. **Find Release PipelineRun (Optional)**
   ```bash
   oc project rhtap-releng-tenant
   oc get pipelineruns -l release.appstudio.openshift.io/name={release-name}
   ```

3. **View Pipeline Logs**
   ```bash
   oc logs -f {pipelinerun-pod-name}
   ```

   Or use the Konflux UI to view detailed task logs.

4. **Common Pipeline Tasks**
   - `verify-enterprise-contract`: Validates against production EC policy
   - `push-snapshot`: Pushes images to target registry
   - `sign-images`: Signs images with Red Hat keys
   - `update-pyxis`: Updates container metadata in Pyxis

### Step 7: Verify Release Success

1. **Check Pipeline Completion**
   ```bash
   oc get pipelinerun {pipelinerun-name}
   ```
   - Status should be `Succeeded`

2. **Verify Images in Registry**
   - For staging:
     ```bash
     podman pull registry.stage.redhat.io/rhcl-1/rhcl-rhel9-operator:1.4.4
     ```
   - For production:
     ```bash
     podman pull registry.redhat.io/rhcl-1/rhcl-rhel9-operator:1.4.4
     ```

3. **Check Pyxis Metadata**
   - Go to: https://pyxis.engineering.redhat.com/
   - Search for repository: `rhcl-1/rhcl-rhel9-operator`
   - Verify new tags are listed

### Step 8: Repeat for Other Components

RHCL releases multiple components together. Repeat the process for:
- Authorino operator and bundle
- DNS operator and bundle (and CoreDNS)
- Limitador operator and bundle
- MCP Gateway
- Supporting images (wasm-shim, console-plugin, developer-portal-controller)
- 1.5 only: Helm charts (`rhcl-1-5-helm-charts-*`) and the RHDH dynamic plugin (`rhcl-1-5-rhcl-rhdh-dynamic-plugin-*`)

See [1.4 vs 1.5](#14-vs-15-how-the-operators-are-delivered) for which bundles matter in each stream.

Each component has its own RPA file in the `api-management` directory.

---

## Step-by-Step: Releasing the File-Based Catalog

The file-based catalog (FBC) publishes operator metadata to OpenShift OperatorHub indexes.

### What is a File-Based Catalog?

A file-based catalog is a YAML file (`catalog.yaml`) that defines:
- **Packages**: Operator packages (e.g., `rhcl-operator`, `dns-operator`)
- **Channels**: Update channels (e.g., `stable`, `fast`)
- **Bundles**: References to operator bundle images
- **Deprecations**: Deprecated versions or channels

### FBC Architecture for RHCL

RHCL maintains separate FBC components for each supported OCP version:
- **OCP 4.14**: `rhcl-file-based-catalog-4-14`
- **OCP 4.15**: `rhcl-file-based-catalog-4-15`
- **OCP 4.16**: `rhcl-file-based-catalog-4-16`
- **OCP 4.17**: `rhcl-file-based-catalog-4-17`
- **OCP 4.18**: `rhcl-file-based-catalog-4-18`
- **OCP 4.19**: `rhcl-file-based-catalog-4-19`
- **OCP 4.20**: `rhcl-file-based-catalog-4-20`
- **OCP 4.21**: `rhcl-file-based-catalog-4-21`
- **OCP 4.22**: `rhcl-file-based-catalog-4-22`

Each OCP version has a `catalog/<ocp-version>/` directory with one sub-directory per operator package (`rhcl-operator`, `dns-operator`, `limitador-operator`, `authorino-operator`, `mcp-gateway-operator`), each holding a `catalog.yaml`. Current releases only add bundles to the newest OCP versions the release supports (see the support matrix below).

### Step 1: Update FBC Catalog YAML

> **Which packages and channels (RHCL 1.4 and 1.5):**
> - **1.4 patches:** add each released operator's bundle to its package in the **`stable-1.4`** channel.
> - **1.5:** add **only** the `rhcl-operator` bundle, to the **`stable-1.5`** channel. Component operators are delivered through the RHCL operator and need no catalog entries.
> - The old `stable` channel is frozen; do not add new bundles to it. If the target channel does not yet exist in a package's `catalog.yaml`, create it (as was done for `stable-1.4` in the rhcl-operator package).
> - The snippets below were written for the old model. Adjust the channel name in the `yq` command and loop only over the packages that apply.

**Important**: FBC catalogs must be updated for each operator across all supported OCP versions.

> Note: See [Supported Configurations for Red Hat Connectivity Link](https://access.redhat.com/articles/7092611) for the support matrix

The update process is identical for all operators in their respective supported versions.

> **Recommended: use the `/add-bundle` skill.** The best way to add FBC content is the Claude Code command in the `rhcl-file-based-catalog` repo (`.claude/commands/add-bundle.md`). Open Claude Code in that repo and run:
> ```
> /add-bundle <BUNDLE_IMAGE_PULLSPEC> <OCP_VERSIONS> <UPGRADE_PATH>
> ```
> Example: `/add-bundle registry.redhat.io/rhcl-1/rhcl-operator-bundle@sha256:abc123 4.20-4.22 replaces 1.5.0 in channel stable-1.5`
>
> It renders the bundle with `opm`, adds the bundle and channel entry for each OCP version, creates a minimal catalog where an operator is new to an OCP version, and runs `opm validate`. It also keeps `authorino-operator` carrying its full catalog history, which the manual steps below do not handle for you.
>
> Tips:
> - Use the bundle pullspec from the `registry.redhat.io` release output (the `Get Bundle URL from Release Snapshot` step below), not the `quay.io/redhat-user-workloads` one.
> - **State the channel explicitly** (`stable-1.4` or `stable-1.5`). The skill's prompts default to the old `stable` channel.
> - Review the diff before committing (`git diff catalog/`); each operator file should change by one bundle and one channel entry.
>
> The manual steps below show what the skill does and are useful for debugging.

**Prerequisites**: Complete component releases first (bundles must exist in registry) and install required tools: `kubectl`, `yq`, `opm`

1. **Navigate to FBC Repository**
   ```bash
   cd /path/to/rhcl-file-based-catalog
   ```

2. **Get Bundle URL from Release Snapshot**

   First, set the Release name. Release names follow the pattern: `rhcl-{version}-{component}-{environment}-rhel9`

   ```bash
   # Set release and operator parameters
   OPERATOR=limitador-operator
   RELEASE="rhcl-1-4-4-limitador-rc1-prod-rhel9"

   # Extract bundle URL from release, it has to be the registry.redhat.io one
   BUNDLE_NAME=rhcl-1-4-${OPERATOR}-bundle
   BUNDLE_SHASUM=$(kubectl get releases ${RELEASE}  -o yaml | NAME=${BUNDLE_NAME} yq e '.status.artifacts.images[] | select(.name == strenv(NAME)).shasum')
   BUNDLE_REGISTRY=$(kubectl get releases ${RELEASE}  -o yaml | yq e '.status.artifacts.filtered_snapshot' | NAME=${BUNDLE_NAME}  yq e '.components[] | select(.name == strenv(NAME)).rh-registry-repo')
   BUNDLE="${BUNDLE_REGISTRY}@${BUNDLE_SHASUM}"
   echo "Bundle ${BUNDLE}"
   ```

3. **Render Bundle and Update All OCP Catalogs**
   ```bash
   # Render the bundle to a temporary file
   TMPFILE=$(mktemp)
   opm render --migrate ${BUNDLE} -o yaml > $TMPFILE

   # Define OCP versions to update
   declare -a ocpversions=("4.19" "4.20" "4.21" "4.22")

   # Loop through each OCP version
   for version in "${ocpversions[@]}"
   do
      CATALOG_FILE=catalog/$version/$OPERATOR/catalog.yaml
      echo "Updating catalog file $CATALOG_FILE"

      # Add new line and append bundle definition
      echo "" >> ${CATALOG_FILE}
      cat $TMPFILE >> ${CATALOG_FILE}

      # Show existing bundles
      echo "Existing bundles in OCP $version:"
      yq e 'select(.schema == "olm.bundle").name' ${CATALOG_FILE}

      # Add channel entry with "replaces" semantics (use stable-1.5 and no replaces target for a new 1.5 stream)
      REPLACE=${OPERATOR}.v1.4.3
      NAME="${OPERATOR}.v1.4.4" REPLACE=${REPLACE} CHANNEL=stable-1.4 yq eval --inplace '(select(.schema == "olm.channel" and .name == strenv(CHANNEL)).entries) += {"name": strenv(NAME), "replaces": strenv(REPLACE)}' $CATALOG_FILE

      # Validate the updated catalog
      opm validate catalog/$version/${OPERATOR}
   done

   # Clean up
   rm $TMPFILE
   ```

4. **Repeat for Remaining Operators**

   Repeat steps 2-3 for the other three operators, adjusting:
   - Release name (e.g., `rhcl-1-4-dns-operator-prod-rhel9`)
   - Bundle component name (e.g., `rhcl-1-4-dns-operator-bundle`)
   - Catalog path (e.g., `catalog/$version/dns-operator/catalog.yaml`)
   - Version in `yq` command (e.g., `dns-operator.v1.4.4`)

5. **Commit and Push Changes**
   ```bash
   git add catalog/
   git commit -m "Update FBC catalogs for RHCL 1.4.4

   - authorino-operator: v1.4.4
   - dns-operator: v1.4.4
   - limitador-operator: v1.4.4
   - rhcl-operator: v1.4.4"
   git push origin main
   ```

### Step 2: Wait for FBC Build

1. **Monitor PipelineRuns**
   ```bash
   oc project api-management-tenant
   oc get pipelineruns -l appstudio.openshift.io/component=rhcl-file-based-catalog-4-19 --sort-by=.metadata.creationTimestamp
   ```

2. **Verify Build Success**
   - Each OCP version FBC component builds separately

### Step 3: Update FBC ReleasePlanAdmission

1. **Locate FBC RPA**
   ```bash
   cd konflux-release-data
   vim config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-file-based-catalog-stage.yaml
   ```

2. **Review FBC Configuration**
   ```yaml
   spec:
     applications: [rhcl-file-based-catalog]
     data:
       fbc:
         fromIndex: 'registry-proxy.engineering.redhat.com/rh-osbs/iib-pub-pending:{{ OCP_VERSION }}'
         targetIndex: ''  # Empty for staging; populated for production
         publishingCredentials: 'staged-index-fbc-publishing-credentials'
         allowedPackages: ["rhcl-operator", "dns-operator", "limitador-operator", "authorino-operator", "mcp-gateway"]
         buildTimeoutSeconds: 1500
         requestTimeoutSeconds: 1500
   ```

3. **For Production Release**
   - Use the separate `rhcl-file-based-catalog-prod.yaml` file. It already carries the production values below; check them rather than editing the stage file:
     ```yaml
     fromIndex: 'registry-proxy.engineering.redhat.com/rh-osbs/iib-pub:{{ OCP_VERSION }}'
     ```
   - Set `targetIndex`:
     ```yaml
     targetIndex: 'quay.io/redhat-prod/redhat----redhat-operator-index:{{ OCP_VERSION }}'
     ```
   - Update `publishingCredentials`:
     ```yaml
     publishingCredentials: 'fbc-production-publishing-credentials-redhat-prod'
     ```
   - Update `serviceAccountName`:
     ```yaml
     pipeline:
       serviceAccountName: release-index-image-prod
     ```

4. **Commit and Create MR**
   ```bash
   git add config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/
   git commit -m "Update FBC RPA for RHCL 1.4.4 production"
   git push origin update-rhcl-fbc-1.4.4
   # Create merge request in GitLab
   ```

### Step 4: Create FBC Release

**Note**: Since all OCP version catalogs are components of the same `rhcl-file-based-catalog` application, a single release covers all OCP versions.

1. **Find FBC Snapshot**
   ```bash
   oc project api-management-tenant
   oc get snapshots -l appstudio.openshift.io/application=rhcl-file-based-catalog --sort-by=.metadata.creationTimestamp
   ```

2. **Create Release**
   ```bash
   cat > rhcl-fbc-release.yaml <<EOF
   apiVersion: appstudio.redhat.com/v1alpha1
   kind: Release
   metadata:
     name: rhcl-fbc-1.4.4-$(date +%s)
     namespace: api-management-tenant
     labels:
       release.appstudio.openshift.io/author: 'your-username'
   spec:
     releasePlan: rhcl-file-based-catalog-stage  # Or production plan
     snapshot: {snapshot-name}
   EOF
   ```

3. **Apply Release**
   ```bash
   oc apply -f rhcl-fbc-release.yaml
   ```

### Step 5: Monitor FBC Release Pipeline

1. **Check Release Status**
   ```bash
   oc get release rhcl-fbc-1.4.4-* -w
   ```

2. **View Release Pipeline in Managed Namespace**
   ```bash
   oc project rhtap-releng-tenant
   oc get pipelineruns -l release.appstudio.openshift.io/name=rhcl-fbc-1.4.4-*
   ```

3. **Monitor IIB Submission**
   - The `fbc-release` pipeline submits the FBC fragment to IIB
   - IIB tasks:
     - **create-fbc-fragment**: Validates and processes catalog YAML
     - **add-to-index**: Merges fragment into existing index
     - **build-index**: Builds new index image
     - **push-index**: Publishes index to target registry


### Step 6: Verify FBC Publication

1. **Check Index Image**
   - For staging:
     ```bash
     podman pull registry-proxy.engineering.redhat.com/rh-osbs/iib-pub-pending:v4.19
     ```
   - For production:
     ```bash
     podman pull quay.io/redhat-prod/redhat----redhat-operator-index:v4.19
     ```

2. **Inspect Index Contents and Verify Channel Entries**

   Verify that the new operator version appears in the catalog channels:

   ```bash
   # Set the operator package name and OCP version
   OPERATOR_PACKAGE="authorino-operator"  # Or dns-operator, limitador-operator, rhcl-operator
   OCP_VERSION="v4.19"

   # Render the index and save to a temporary file
   TMPFILE=$(mktemp)
   opm render registry.redhat.io/redhat/redhat-operator-index:${OCP_VERSION} -o json > $TMPFILE

   # Verify channel entries for the operator
   jq -s 'map(select(.package == "'"${OPERATOR_PACKAGE}"'" and .schema == "olm.channel")) | reduce .[] as $channel ({}; .[$channel.name] = [$channel.entries[].name])' $TMPFILE

   # Clean up
   rm $TMPFILE
   ```

   **Expected output:**
   ```json
   {
     "stable-1.4": [
       "authorino-operator.v1.4.2",
       "authorino-operator.v1.4.3",
       "authorino-operator.v1.4.4"
     ]
   }
   ```

   Verify that your new version (e.g., `v1.4.4`) appears in the channel entry list.

3. **Test in OpenShift**
   - Install OpenShift 4.19 (or target version)
   - Check OperatorHub:
     ```bash
     oc get packagemanifests | grep rhcl
     ```
   - Verify new version appears in the web console

---

---

## Patch Releases

A patch release (e.g. 1.4.4, 1.5.1) ships fixes on an existing stream. It reuses the stream's applications, components, RPAs, and Pyxis repositories, so there is **no new Konflux setup**. Pick the section that matches the stream.

> Each RHCL patch release is the functional equivalent of the matching upstream patch release. This guide covers only how to build and ship it through Konflux, not what the upstream patch contains.

### Before you start (both streams)

1. Create the Jira productization issue for the patch with its subtasks (see [Prerequisites](#prerequisites)).
   Decide now whether the patch contains security fixes. If it does, collect the CVE identifiers and the Konflux components they affect, because the Release must be an `RHSA` with a `cves` list.
2. Confirm the stream branch exists and builds are green: `rhcl-1.4` or `rhcl-1.5` in each `*-product-build` repo.
3. Identify the upstream patch tags that correspond to this release. The daily bump workflow may already have opened PRs for them; otherwise bump manually.

### Patch release of 1.4 (separate bundles)

In 1.4 every operator is released and cataloged on its own, so a patch touches each operator that changed. Reference: the 1.4.3 and 1.4.4 patches.

1. **Bump the build repos** (branch `rhcl-1.4`). For each component with changes, open a PR to `rhcl-1.4`:
   - Move the submodule to the new upstream tag and update the `version` label in the Containerfile
   - In operator repos, bump the bundle version in `bundle-generation/*.yaml` (e.g. `1.4.4`)
   - Merge the PR. The *Generate Bundles* workflow commits regenerated manifests, and nudge PRs flow into the bundles. Merge those too.
   - Example: [`bump-authorino-operator-1.4.4`](https://github.com/rh-api-management/authorino-operator-product-build/pull/310)
2. **Wait for a clean Snapshot** and run [Step 1](#step-1-verify-builds-are-complete): each bundle's CSV must reference the operand digests in that Snapshot.
3. **Update RPA tags** in `konflux-release-data` (one MR, both `-stage-` and `-prod-` files for each changed component under `ReleasePlanAdmission/api-management/rhcl-1-4-*`):
   - Change the patch tag, e.g. `rhcl-1.4.3` → `rhcl-1.4.4`
   - Image version tags now come from the image labels (`{{labels.version}}`). Only components with hard-coded versions (coredns) need their tags edited.
   - Run `tox`, then merge. Example MR: [!22121 "Tags for RHCL 1.4.3"](https://gitlab.cee.redhat.com/releng/konflux-release-data/-/merge_requests/22121)
4. **Release to stage, then prod** for each changed application, following [Steps 5–7](#step-5-create-a-releaseplan). Set the Release `type` to `RHSA` and list the `cves` if the patch has any security fix, otherwise `RHBA` ([Step 6](#step-6-create-a-release)). Reuse the existing ReleasePlans (they live in `rhcl-releaseplans/rhcl-1-4/`); create new ones only if the advisory text must change.
5. **Update the catalog** with the [`/add-bundle` skill](#step-1-update-fbc-catalog-yaml) ([FBC guide](#step-by-step-releasing-the-file-based-catalog)). For each released operator, add the new bundle to OCP 4.19–4.22 and add it to the **`stable-1.4`** channel (not the old `stable` channel), with `replaces` pointing at the previous 1.4.x. Use the same release you just made as the bundle source. Example: `Add rhcl-operator v1.4.4 to new stable-1.4 channel in OCP 4.19-4.22`.
6. **Release the FBC** application (stage first, then prod).
7. **Verify** the images in Pyxis and the new version in the index ([Step 6: Verify FBC Publication](#step-6-verify-fbc-publication)).

### Patch release of 1.5 (consolidated through the RHCL operator)

In 1.5 the RHCL operator deploys all component operators from charts built into its image. A patch is therefore driven through `rhcl-operator-product-build`, and the catalog only needs the RHCL operator bundle.

> There has not been a 1.5.x patch yet, so the build and RPA steps are based on how 1.5 is designed. Update this section after the first one.

1. **Bump the component repos** (branch `rhcl-1.5`): authorino, limitador, dns-operator, mcp-gateway, wasm-shim, console plugin, etc. Merge the submodule bump PRs for the components that have fixes.
2. **Let the nudges reach `rhcl-operator`.** Each component build opens a nudge PR in `rhcl-operator-product-build` that updates `bundle-generation/image-pullspecs/*.yaml`. The *Generate Component Charts*, *Generate Helm Chart* and *Generate Bundles* workflows then commit the regenerated charts and bundle manifests. Merge those PRs and wait for the build to finish.
3. **Bump the RHCL operator version** if there is an operator change or a new patch number: `bundle-generation/rhcl-operator.yaml` (CSV name/version) and the version labels in the `Containerfile.rhcl-operator*` files. Keep the channel at `stable-1.5`.
4. **Verify the Snapshot** for `rhcl-1-5-rhcl-operator`: the bundle CSV `relatedImages` and the component charts must reference the operand digests you intend to ship ([Step 1](#step-1-verify-builds-are-complete)). If you updated an operand, check its digest in the CSV rather than assuming the nudge landed.
5. **Update RPA tags** in `konflux-release-data` for what you are releasing: `rhcl-1-5-rhcl-operator-*`, the component RPAs (`rhcl-1-5-authorino-operator-*`, `dns-operator`, `limitador-operator`, `mcp-gateway`, `coredns`), `rhcl-1-5-helm-charts-*`, and `rhcl-1-5-rhcl-rhdh-dynamic-plugin-*` if they changed. Change the patch tags (`rhcl-1.5.0` → `rhcl-1.5.1`, `v1.5.1`, `1.5.1`). Run `tox` and merge.
6. **Release to stage, then prod** (`RHSA` with `cves` if there are security fixes, otherwise `RHBA`, see [Step 6](#step-6-create-a-release)): the RHCL operator application first, then the component and helm chart applications that changed, using the existing ReleasePlans in `rhcl-releaseplans/rhcl-1-5/`.
7. **Update the catalog** with the [`/add-bundle` skill](#step-1-update-fbc-catalog-yaml): add only the new `rhcl-operator` bundle to OCP 4.20–4.22 (the 1.5 support range), in the **`stable-1.5`** channel with `replaces` the previous 1.5.x. Component operators (authorino, dns, limitador, mcp-gateway) get **no** catalog entries. Then release the FBC application.
8. **Verify** in Pyxis and the index as for 1.4.

### Key differences at a glance

| Step | 1.4 | 1.5 |
|------|-----|-----|
| Where fixes enter | Each operator repo and its bundle | `rhcl-operator-product-build` (via nudges and generated charts) |
| Bundles to check | One per changed operator | RHCL operator bundle |
| RPAs to edit | Per changed component | RHCL operator, changed components, helm charts, dynamic plugin |
| Catalog entries | One per changed operator, in `stable-1.4` | RHCL operator only, in `stable-1.5` |

### Getting Help

1. **Konflux Users Slack**
   - Channel: #konflux-users
   - Use "Ask for Support" button for tickets

2. **Konflux Release Slack**
   - Channel: #forum-konflux-release
   - For release-specific questions

3. **Konflux Releng Team**
   - Tag `@konflux-releng` in #konflux-users or MR comments
   - For RPA reviews and approvals

---

## Additional Resources

### Documentation

- **Konflux Docs**: https://konflux.pages.redhat.com/docs/users
- **Release Data Repo**: https://gitlab.cee.redhat.com/releng/konflux-release-data
- **Preparing for Release**: https://konflux.pages.redhat.com/docs/users/releasing/preparing-for-release.html
- **Releasing Products**: https://konflux.pages.redhat.com/docs/users/releasing/releasing-products.html
- **Enterprise Contract Policies**: https://enterprisecontract.dev/docs/ec-policies/
- **Operator Lifecycle Manager (OLM)**: https://olm.operatorframework.io/

### Internal Resources

- **Release Engineering Workflow**: https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/docs/releng/releng_workflow.md
- **Konflux Release Data README**: https://gitlab.cee.redhat.com/releng/konflux-release-data/-/blob/main/README.md
- **RHCL GitHub Organization**: https://github.com/rh-api-management

### Key Repositories

- **rh-api-management**: https://github.com/rh-api-management (RHCL source)
- **konflux-release-data**: https://gitlab.cee.redhat.com/releng/konflux-release-data (Release config)
- **pyxis-repo-configs**: https://gitlab.cee.redhat.com/releng/pyxis-repo-configs (Registry metadata)
- **release-service-catalog**: https://github.com/konflux-ci/release-service-catalog (Release pipelines)

---

## Appendix: Quick Reference

### Common Commands

```bash
# Login to Konflux
oc login https://api.stone-prd-rh01.pg1f.p1.openshiftapps.com

# Switch to developer workspace
oc project api-management-tenant

# View recent builds
oc get pipelineruns --sort-by=.metadata.creationTimestamp | tail -20

# Check snapshot
oc get snapshot {name} -o yaml

# Create a release
oc apply -f release.yaml

# Monitor release
oc get release {name} -w

# Switch to managed workspace
oc project rhtap-releng-tenant

# View release pipeline
oc get pipelineruns -l release.appstudio.openshift.io/name={release-name}

# Check RPA
oc get releaseplanadmission {name} -o yaml
```

### RPA File Locations

- **Production**: `config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-*-prod-rhel9.yaml`
- **Staging**: `config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-*-stage-rhel9.yaml`
- **FBC**: `config/stone-prd-rh01.pg1f.p1/product/ReleasePlanAdmission/api-management/rhcl-file-based-catalog-*.yaml`
- **Helm charts (1.5)**: `.../api-management/rhcl-1-5-helm-charts-{stage,prod}.yaml`
- **ReleasePlans**: `tenants-config/cluster/stone-prd-rh01/tenants/api-management-tenant/rhcl-releaseplans/`
- **Project streams and templates**: `tenants-config/cluster/stone-prd-rh01/tenants/api-management-tenant/rhcl-project-streams/`
- **Pyxis repos**: [`pyxis-repo-configs`](https://gitlab.cee.redhat.com/releng/pyxis-repo-configs) `products/rhcl/`

### Registry Mappings

| Environment | Registry | Example                                                                        |
|-------------|----------|--------------------------------------------------------------------------------|
| Development | quay.io/redhat-user-workloads/api-management-tenant/ | `quay.io/redhat-user-workloads/api-management-tenant/rhcl-operator:{revision}` |
| Staging | registry.stage.redhat.io | `registry.stage.redhat.io/rhcl-1/rhcl-rhel9-operator:1.4.4`                    |
| Production | registry.redhat.io | `registry.redhat.io/rhcl-1/rhcl-rhel9-operator:1.4.4`                          |

---
