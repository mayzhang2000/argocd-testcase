# argocd-testcase

## The Issue

As far as I can tell, running with namespace isolation in a cluster with a relatively high
number of resource types (CRDs) creates a situation where some changes are
applied to the cluster but the applications that applied those changes remain
incorrectly 'OutOfSync'.

Anecdotally, it seems to me that the number of applications or resources managed by
ArgoCD is relevant to reproducing this issue. It doesn't seem to happen for
me with a couple of applications, but does with six or more.

I have found three 'solutions' to this problem which I believe are relevant:

- Reducing the number of resource types in the cluster by deleting CRDs
- Defining a smaller list of `resource.inclusions` in `argocd-cm`. In my testing,
  including 20 namespaced types out of a possible 50 seems to have resolved the
  issues for me.
- Removing namespaces from the cluster configuration and applying the upstream
  cluster roles

## Steps to reproduce

### Requirements

You need `minikube`, `kustomize` and `kubectl` installed.

These are the versions I'm running:

```
$ minikube version
minikube version: v1.6.1
commit: 42a9df4854dcea40ec187b6b8f9a910c6038f81a
```

```
$ kustomize version
{Version:3.5.4 GitCommit:3af514fa9f85430f0c1557c4a0291e62112ab026 BuildDate:2020-01-11T06:57:29+00:00 GoOs:darwin GoArch:amd64}
```

```
$ kubectl --context=minikube version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2", GitCommit:"52c56ce7a8272c798dbc29846288d7cd9fbae032", GitTreeState:"clean", BuildDate:"2020-04-16T23:35:15Z", GoVersion:"go1.14.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.3", GitCommit:"06ad960bfd03b39c8310aaf92d1e7c12ce618213", GitTreeState:"clean", BuildDate:"2020-02-11T18:07:13Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
```

### Setup

Fork this repository and edit the repoURL in the applications accordingly:

```
$ sed -i 's/ribbybibby/YOUR_USERNAME/g' manifests/my-argocd/applications/*.yaml
$ git add .
$ git commit -m "Edit repoURL"
$ git push origin
```

Start minikube:

```
$ minikube start --kubernetes-version=v1.17.3
😄  minikube v1.6.1 on Darwin 10.15.3
✨  Automatically selected the 'hyperkit' driver (alternates: [virtualbox])
🔥  Creating hyperkit VM (CPUs=2, Memory=2000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.17.3 on Docker '19.03.5' ...
🚜  Pulling images ...
🚀  Launching Kubernetes ...
⌛  Waiting for cluster to come online ...
🏄  Done! kubectl is now configured to use "minikube"
```

Apply the manifests from the root of the repository. You may need to run this twice.

```
$ kustomize build manifests | kubectl --context=minikube apply -f -
```

Port-forward the UI from the server pod:

```
$ kubectl --context=minikube -n my-argocd get pods
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-6758b7fdcd-64g96   1/1     Running   0          6m46s
argocd-dex-server-7b5bd689fb-pqt2r               1/1     Running   0          6m45s
argocd-redis-8c568b5db-j4kxd                     1/1     Running   0          6m45s
argocd-repo-server-59fddff8f8-4d5qv              1/1     Running   0          6m45s
argocd-server-57b799445d-j664g                   1/1     Running   0          6m45s
$ kubectl --context=minikube -n my-argocd port-forward argocd-server-57b799445d-j664g 8080
```

Visit https://localhost:8080 in your browser and login as `admin` with password `<argocd-server pod name>`.

### Observing the issue

Some of the apps in the UI may already be showing as OutOfSync, which is
indicative of the issue.

For further confirmation, make a change, commit it and push:

```
find . -type f -name echo.txt -exec sed -i 's/Lorem/Foobar/g' {} \;
git add manifests/
git commit -m "Lorem -> Foobar"
git push origin
```

Wait for the applications in the UI to automatically sync. Observe that after
syncing some of the applications remain OutOfSync. If you examine the objects
that it claims are out of sync with `kubectl get -o yaml` you should find that they have
been applied correctly.

Note that restarting the controller will return all applications to a 'Synced'
state. You may have to wait a few minutes for that to happen.

```
kubectl --context=minikube -n my-argocd rollout restart deployment argocd-application-controller
```

Try another change:

```
sed -i 's/replicas: 1/replicas: 2/g' manifests/my-echo-{3,6}/echo.yaml
git add manifests/
git commit -m "Increase number of replicas for 3 and 6"
git push origin
```

Observe the same OutOfSync behaviour.

### Resolving the issue

Delete the crds in `manifests/kube-system/crds.yaml`:

```
kubectl --context=minikube delete -f manifests/kube-system/crds.yaml
```

Observe that all the applications are now reporting as 'Synced' in the UI, if
they weren't before.

Make some more changes:

```
find . -type f -name echo.txt -exec sed -i 's/Foobar/Barfoo/g' {} \;
git add manifests/
git commit -m "Foobar -> Barfoo"
git push origin
```

Note in the UI that these changes are synced as expected. Continue to make
changes and you should observe that Argo syncs resources as expected.
