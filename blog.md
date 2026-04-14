# Taming Complex ApplicationSets with the Rendered Manifests Pattern

ApplicationSet is one of the most powerful features in Argo CD. It enables platform administrators to automate Application management without writing code — just declarative YAML that generates Applications dynamically based on your environment.

## What ApplicationSet Does Well

The simplest use cases are compelling on their own:

- **One app per directory**: point an ApplicationSet at a Git repository, and it creates an Application for every directory it discovers. Add a new microservice, get a new deployment automatically.
- **Apps per cluster**: define a template, and every cluster registered in Argo CD gets the same set of applications. Onboard a new cluster, and it's fully configured within minutes.

These patterns replace hundreds of lines of copy-pasted Application manifests with a single ApplicationSet definition.

## When Things Get Complex

Real-world infrastructure rarely fits into simple patterns. Consider cluster addon management: you need a base set of addons deployed to every cluster, but production clusters need different resource limits than staging. Some clusters are "snowflakes" — they run specialized workloads that require unique configurations.

A project like [argocd-core-cluster-management](https://github.com/alexmt/argocd-core-cluster-management) demonstrates how to solve this with ApplicationSet. It uses a layered directory structure with matrix and merge generators to handle:

- **Base addons** deployed to all clusters (Traefik, Grafana, monitoring)
- **Cluster groups** for environment-specific overrides (prod, stage, test)
- **Snowflake clusters** with per-cluster customizations

The result is elegant — but the ApplicationSet definition that drives it is not trivial. It combines multiple generators, relies on cluster labels for targeting, and a single misconfiguration can affect every cluster in your fleet.

## The Problem: Complexity Meets Critical Infrastructure

This is where things get uncomfortable. ApplicationSets that manage cluster addons or core infrastructure are both:

1. **Complex** — matrix generators, merge generators, Go templating, label selectors, multiple git directories
2. **Critical** — a bad template can roll out a broken configuration to every production cluster simultaneously

That's not a great combination. A typo in a generator template or an unintended label match can cause cascading failures across your entire fleet. And because ApplicationSet evaluates dynamically, you don't see the generated Applications until they're already being synced.

Traditional pull request reviews help, but reviewing an ApplicationSet diff doesn't tell you *what Applications will actually be generated*. You're reviewing the template, not the output.

## The Rendered Manifests Pattern

What if you could review the actual generated Applications before they reach your clusters?

The rendered manifests pattern separates ApplicationSet evaluation from deployment:

1. **Define** your ApplicationSet as usual — generators, templates, all the complexity you need
2. **Render** the ApplicationSet in CI using `argocd appset generate`, producing concrete Application manifests
3. **Store** the rendered manifests in a dedicated Git branch (e.g., `rendered`)
4. **Deploy** using a simple app-of-apps that reads from the `rendered` branch

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│  Edit AppSet │────▶│  CI Pipeline  │────▶│ rendered branch │────▶│ app-of-apps  │
│  on main     │     │  appset       │     │ (concrete apps) │     │ syncs to     │
│              │     │  generate     │     │                 │     │ cluster      │
└─────────────┘     └──────────────┘     └─────────────────┘     └──────────────┘
```

### What This Gives You

**Reviewable output.** When someone changes the ApplicationSet template or the app manifests, the CI pipeline regenerates the concrete Applications. The pull request to the `rendered` branch shows you exactly which Applications will be created, modified, or deleted. No more guessing what a generator change will produce.

**Git history of generated state.** Every change to the rendered branch is a commit with a clear diff. You can trace exactly when an Application was added or modified, and correlate it back to the source change on `main`.

**No ApplicationSet controller in the cluster.** The cluster only sees plain Application resources from the `rendered` branch. This reduces the attack surface and eliminates a class of runtime errors where the ApplicationSet controller misinterprets a template or encounters a transient API error during generation.

**Safe rollback.** If a rendered change causes problems, revert the commit on the `rendered` branch. The app-of-apps will sync back to the previous state immediately.

### The Trade-off

You lose real-time dynamic generation. If a new cluster is added, the CI pipeline needs to run before Applications are created for it. For most teams managing infrastructure at scale, this is a feature, not a bug — you want changes to go through CI, not happen automatically in the background.

## Try It

We've published a complete working example at [alexmt/appset-rendered-manifests](https://github.com/alexmt/appset-rendered-manifests). It includes:

- A Kustomize-based Argo CD installation
- An ApplicationSet with a matrix generator (clusters × apps)
- Sample applications
- A GitHub Actions workflow that renders and pushes to an orphan branch
- An app-of-apps that deploys from the rendered branch

Clone it, swap in your own clusters and apps, and you have a production-ready setup for managing complex ApplicationSets safely.

## Conclusion

ApplicationSet is incredibly powerful — powerful enough to manage your entire fleet's infrastructure from a single definition. But with that power comes risk. The rendered manifests pattern gives you a safety net: full visibility into what will be deployed, a reviewable Git history, and the confidence that what you see in the PR is exactly what reaches your clusters.

Stop reviewing templates. Start reviewing output.
