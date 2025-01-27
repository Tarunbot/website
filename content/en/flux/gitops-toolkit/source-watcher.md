---
title: "Watching for source changes"
linkTitle: "Watching for source changes"
description: "Develop a Kubernetes controller that reacts to source changes."
weight: 20
---

In this guide you'll be developing a Kubernetes controller with
[Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
that subscribes to [GitRepository](../components/source/gitrepositories.md)
events and reacts to revision changes by downloading the artifact produced by
[source-controller](../components/source/_index.md).

## Prerequisites

On your dev machine install the following tools:

* go >= 1.19
* kubebuilder >= 3.0
* kind >= 0.8
* kubectl >= 1.21

## Install Flux

Install the Flux CLI with Homebrew on macOS or Linux:

```sh
brew install fluxcd/tap/flux
```

Create a cluster for testing:

```sh
kind create cluster --name dev
```

Verify that your dev machine satisfies the prerequisites with:

```sh
flux check --pre
```

Install source-controller on the dev cluster:

```sh
flux install \
--namespace=flux-system \
--network-policy=false \
--components=source-controller
```

## Clone the sample controller

You'll be using [fluxcd/source-watcher](https://github.com/fluxcd/source-watcher) as
a template for developing your own controller. The source-watcher was scaffolded with `kubebuilder init`.

Clone the source-watcher repository:

```sh
git clone https://github.com/fluxcd/source-watcher
cd source-watcher
```

Build the controller:

```sh
make
```

## Run the controller

Port forward to source-controller artifacts server:

```sh
kubectl -n flux-system port-forward svc/source-controller 8181:80
```

Export the local address as `SOURCE_CONTROLLER_LOCALHOST`:

```sh
export SOURCE_CONTROLLER_LOCALHOST=localhost:8181
```

Run source-watcher locally:

```sh
make run
```

Create a Git source:

```sh
flux create source git test \
--url=https://github.com/fluxcd/flux2 \
--ignore-paths='/*,!/manifests' \
--tag=v0.34.0
```

The source-watcher will log the revision:

```sh
New revision detected   {"gitrepository": "flux-system/test", "revision": "v0.34.0/90f0d81532f6ea76c30974267956c7eaee5c1dea"}
Processing manifests...
```

Change the Git tag:

```sh
flux create source git test \
--url=https://github.com/fluxcd/flux2 \
--ignore-paths='/*,!/manifests' \
--tag=v0.35.0
```

And source-watcher will log the new revision:

```sh
New revision detected   {"gitrepository": "flux-system/test", "revision": "v0.35.0/1bf63a94c22d1b9b7ccf92f66a1a34a74bd72fca"}
```

The source-controller reports the revision under `GitRepository.Status.Artifact.Revision` in the format: `<branch|tag>/<commit>`.

## How it works

The [GitRepositoryWatcher](https://github.com/fluxcd/source-watcher/blob/main/controllers/gitrepository_watcher.go)
controller does the following:

* subscribes to `GitRepository` events
* detects when the Git revision changes
* downloads and extracts the source artifact
* writes the extracted dir names to stdout

```go
type GitRepositoryWatcher struct {
	client.Client
	HttpRetry       int
	artifactFetcher *fetch.ArchiveFetcher
}

func (r *GitRepositoryWatcher) SetupWithManager(mgr ctrl.Manager) error {
	r.artifactFetcher = fetch.NewArchiveFetcher(
		r.HttpRetry,
		tar.UnlimitedUntarSize,
		os.Getenv("SOURCE_CONTROLLER_LOCALHOST"),
	)

	return ctrl.NewControllerManagedBy(mgr).
		For(&sourcev1.GitRepository{}, builder.WithPredicates(GitRepositoryRevisionChangePredicate{})).
		Complete(r)
}

// +kubebuilder:rbac:groups=source.toolkit.fluxcd.io,resources=gitrepositories,verbs=get;list;watch
// +kubebuilder:rbac:groups=source.toolkit.fluxcd.io,resources=gitrepositories/status,verbs=get

func (r *GitRepositoryWatcher) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrl.LoggerFrom(ctx)

	// get source object
	var repository sourcev1.GitRepository
	if err := r.Get(ctx, req.NamespacedName, &repository); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	artifact := repository.Status.Artifact
	log.Info("New revision detected", "revision", artifact.Revision)

	// create tmp dir
	tmpDir, err := os.MkdirTemp("", repository.Name)
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("failed to create temp dir, error: %w", err)
	}
	defer os.RemoveAll(tmpDir)

	// download and extract artifact
	if err := r.artifactFetcher.Fetch(artifact.URL, artifact.Checksum, tmpDir); err != nil {
		log.Error(err, "unable to fetch artifact")
		return ctrl.Result{}, err
	}

	// list artifact content
	files, err := os.ReadDir(tmpDir)
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("failed to list files, error: %w", err)
	}

	// do something with the artifact content
	for _, f := range files {
		log.Info("Processing " + f.Name())
	}

	return ctrl.Result{}, nil
}
```

To add the watcher to an existing project, copy the controller and the revision change predicate to your `controllers` dir:

* [gitrepository_watcher.go](https://github.com/fluxcd/source-watcher/blob/main/controllers/gitrepository_watcher.go)
* [gitrepository_predicate.go](https://github.com/fluxcd/source-watcher/blob/main/controllers/gitrepository_predicate.go)

In your `main.go` init function, register the Source API schema:

```go
import (
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"

	sourcev1 "github.com/fluxcd/source-controller/api/v1beta2"
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))
	utilruntime.Must(sourcev1.AddToScheme(scheme)

	// +kubebuilder:scaffold:scheme
}
```

Start the controller in the main function:

```go
func main()  {

	if err = (&controllers.GitRepositoryWatcher{
		Client:    mgr.GetClient(),
		HttpRetry: httpRetry,
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "GitRepositoryWatcher")
		os.Exit(1)
	}

}
```

Note that the watcher depends on Flux [runtime](https://pkg.go.dev/github.com/fluxcd/pkg/runtime)
and Kubernetes [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime):

```go
require (
    github.com/fluxcd/pkg/runtime v0.20.0
    sigs.k8s.io/controller-runtime v0.13.0
)
```

That's it! Happy hacking!
