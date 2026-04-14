# Taming Complex ApplicationSets with the Rendered Manifests Pattern

I've been working on Argo CD for a long time, and ApplicationSet is one of those features that keeps surprising me. The simple use cases are easy — point it at a Git repo and get an Application for every directory, or stamp out the same apps across every cluster. Most people start there and it works great.

But then reality kicks in.

## It Gets Complex Fast

Take cluster addon management. You want a base set of addons on every cluster, but prod needs different resource limits than staging. And then there's that one snowflake cluster running GPU workloads that needs its own config.

I put together [argocd-core-cluster-management](https://github.com/alexmt/argocd-core-cluster-management) a while back to show how you can handle this with ApplicationSet — layered directories, matrix generators, merge generators, cluster groups, the works. It's a clean setup and it works well.

But here's the thing: the ApplicationSet definition that makes it all work is not simple. It's got matrix generators nested with git generators, label selectors, Go templates. And this thing is managing your core infrastructure — addons, monitoring, networking across your entire fleet.

## The Uncomfortable Part

Complex + critical = scary. I've seen this pattern play out:

- Someone updates a generator template
- PR review looks reasonable — the YAML diff makes sense
- It gets merged
- Turns out it matched clusters it wasn't supposed to
- Now every prod cluster is getting a config it shouldn't have

The problem is you're reviewing the *template*, not the *output*. You can't tell from a generator diff what Applications will actually be created. You're trusting the template logic, and template logic is where bugs hide.

## What If You Could See the Output First?

This is what I've been calling the "rendered manifests" pattern. The idea is simple:

1. Write your ApplicationSet as usual — go wild with generators
2. In CI, run `argocd appset generate` to render the actual Application manifests
3. Push those rendered manifests to a separate git branch
4. Have a plain app-of-apps sync from that branch

```
 Edit AppSet  →  CI renders  →  rendered branch  →  app-of-apps syncs
 (on main)       concrete        (reviewable!)       to cluster
                 Applications
```

Now when someone changes the ApplicationSet, CI regenerates everything, and you get a diff of the *actual Applications* that will exist. Not the template — the output. You can see "oh, this change adds an Application to cluster-7 that shouldn't be there" before it ever touches your cluster.

## What You Get

**You can actually review what's changing.** The PR to the rendered branch shows concrete Application specs. No mental template evaluation required.

**Git history of your generated state.** Every rendered commit is traceable back to the source change. When something breaks, `git log` on the rendered branch tells you exactly what changed and when.

**No ApplicationSet controller needed in the cluster.** The cluster just sees plain Applications. One less thing running, one less thing to break.

**Easy rollback.** Revert a commit on the rendered branch and the app-of-apps syncs back. Done.

## The Trade-off

You lose the "automatic" part of ApplicationSet — if you add a new cluster, you need to run the CI pipeline before it gets its apps. Honestly, for anything managing critical infrastructure, I think that's a good thing. You *want* a CI gate there.

## Try It Out

I put together a working example: [alexmt/appset-rendered-manifests](https://github.com/alexmt/appset-rendered-manifests)

It's got everything wired up — Kustomize-based Argo CD install, an ApplicationSet with a matrix generator, sample apps, a GitHub Actions workflow that renders to an orphan branch, and the app-of-apps that ties it together. Fork it and swap in your own stuff.

If you've been burned by an ApplicationSet change that did more than you expected — or you're just nervous about it happening — give this a try. It's a small amount of extra setup for a lot more confidence.

Stop reviewing templates. Start reviewing output.
