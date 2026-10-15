# Helm – Kubernetes Package Manager

Concepts plus a hands-on lab, tested on a kubeadm home-lab cluster (Kubernetes v1.32, Helm v3.21.4).

## What is Helm?

Helm is a **package manager for Kubernetes**. It installs, upgrades, rolls back and deletes applications in a Kubernetes cluster, just like `dnf` on RHEL or `apt` on Ubuntu manage packages on Linux.

## Key terms

| Term | Meaning | Linux comparison |
| --- | --- | --- |
| **Chart** | A package: a bundle of Kubernetes YAML templates for one application | `.rpm` / `.deb` package |
| **Repository** | A place where charts are stored and downloaded from | yum / apt repository |
| **Release** | One installed copy of a chart in the cluster, with its own name | An installed package |
| **Revision** | A numbered version of a release; every install, upgrade or rollback adds one | Package history (`dnf history`) |
| **Values** | Settings that customise a chart (replicas, image, ports, …) | Config file in `/etc` |

## Why use Helm?

### 1. Package management
Instead of applying many YAML files one by one (Deployment, Service, ConfigMap, …), Helm installs the whole application with one command.

### 2. Versioning and rollback
You can install a specific **chart version**:

```bash
helm install my-app podinfo/podinfo --version 6.12.0
```

> The **chart version** is the version of the package. The **app version** is the version of the software inside it. They are often the same, but not always (see the lab output below, e.g. chart `1.4.2` ships app `1.4.1`).

Every install or upgrade creates a new **revision**, so you can go back easily:

```bash
helm history my-app          # list revisions
helm rollback my-app 1       # go back to revision 1
```

### 3. Configuration management
Override settings with a values file:

```bash
helm install my-app podinfo/podinfo -f custom-values.yaml
```

or a single value on the command line:

```bash
helm install my-app podinfo/podinfo --set replicaCount=2
```

## How Helm works

On Linux:

```bash
sudo apt install nginx
```

APT downloads nginx from a repository and installs it on the system. Helm works the same way:

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo   # add the repository
helm repo update                                              # refresh the list (like apt update)
helm install my-app podinfo/podinfo -n helm-lab --create-namespace
```

Helm downloads the **chart** from the repository, fills its templates with the **values**, creates the Kubernetes resources (Deployment, Service, …), and records it as a **release** named `my-app`.

```
Repository ──download──► Chart (templates) + Values ──render──► Kubernetes YAML ──apply──► Release "my-app" (revision 1, 2, 3 …)
```

## Common commands

| Task | Command |
| --- | --- |
| Add a repository | `helm repo add <name> <url>` |
| Refresh repositories | `helm repo update` |
| Search charts | `helm search repo <keyword>` |
| List all chart versions | `helm search repo <keyword> --versions` |
| See default values | `helm show values <repo>/<chart>` |
| Install | `helm install <release> <repo>/<chart> -n <ns>` |
| List releases | `helm list -A` |
| Upgrade / change settings | `helm upgrade <release> <repo>/<chart> -n <ns> -f values.yaml` |
| See current custom settings | `helm get values <release> -n <ns>` |
| History | `helm history <release> -n <ns>` |
| Roll back | `helm rollback <release> <revision> -n <ns>` |
| Uninstall | `helm uninstall <release> -n <ns>` |

---

# Lab – Helm hands-on with podinfo

`podinfo` is a small demo web app with a public Helm chart, which makes it ideal for practising. All output below is real output from the lab cluster.

## Step 1 – Check Helm

```bash
helm version
```

```text
version.BuildInfo{Version:"v3.21.4", GitCommit:"813176c51bb5c181dbbd7901298ddcc104cd3417", GitTreeState:"clean", GoVersion:"go1.26.5"}
```

## Step 2 – Add the repository

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm repo list
```

```text
"podinfo" has been added to your repositories

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "podinfo" chart repository
...Successfully got an update from the "csi-driver-nfs" chart repository
...Successfully got an update from the "argo" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

NAME                    URL
argo                    https://argoproj.github.io/argo-helm
csi-driver-nfs          https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
prometheus-community    https://prometheus-community.github.io/helm-charts
podinfo                 https://stefanprodan.github.io/podinfo
```

## Step 3 – Search charts and versions

```bash
helm search repo podinfo
helm search repo podinfo --versions | head -6
```

```text
NAME            CHART VERSION   APP VERSION     DESCRIPTION
podinfo/podinfo 6.15.0          6.15.0          Podinfo Helm chart for Kubernetes

NAME            CHART VERSION   APP VERSION     DESCRIPTION
podinfo/podinfo 6.15.0          6.15.0          Podinfo Helm chart for Kubernetes
podinfo/podinfo 6.14.1          6.14.1          Podinfo Helm chart for Kubernetes
podinfo/podinfo 6.14.0          6.14.0          Podinfo Helm chart for Kubernetes
podinfo/podinfo 6.13.0          6.13.0          Podinfo Helm chart for Kubernetes
podinfo/podinfo 6.12.0          6.12.0          Podinfo Helm chart for Kubernetes
```

Older entries in the full list show that chart and app versions can differ:

```text
podinfo/podinfo 1.4.2           1.4.1           Podinfo Helm chart for Kubernetes
podinfo/podinfo 0.2.2           0.6.0           Podinfo Helm chart for Kubernetes
```

In this lab, **6.12.0** is the older version to install first and **6.15.0** (newest) is the upgrade target.

## Step 4 – Look at the default values

```bash
helm show values podinfo/podinfo | head -40
```

```yaml
# Default values for podinfo.

replicaCount: 1
logLevel: info
...
image:
  repository: ghcr.io/stefanprodan/podinfo
  tag: 6.15.0
  pullPolicy: IfNotPresent
...
ui:
  color: "#34577c"
  message: ""
  logo: ""
...
service:
  enabled: true
  type: ClusterIP
  metricsPort: 9797
  httpPort: 9898
  externalPort: 9898
  grpcPort: 9999
```

These are the settings that can be overridden with `--set` or `-f`.

## Step 5 – Install an older version

```bash
helm install my-app podinfo/podinfo --version 6.12.0 -n helm-lab --create-namespace
```

```text
NAME: my-app
LAST DEPLOYED: Thu Oct 15 19:05:50 2026
NAMESPACE: helm-lab
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl -n helm-lab port-forward deploy/my-app-podinfo 8080:9898
```

```bash
helm list -n helm-lab
kubectl get all -n helm-lab
```

```text
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
my-app  helm-lab        1               2026-10-15 19:05:50.91905204 +0000 UTC  deployed        podinfo-6.12.0  6.12.0

NAME                                 READY   STATUS    RESTARTS   AGE
pod/my-app-podinfo-ff675469c-hkqfs   1/1     Running   0          33s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/my-app-podinfo   ClusterIP   10.98.211.138   <none>        9898/TCP,9999/TCP   33s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-app-podinfo   1/1     1            1           33s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/my-app-podinfo-ff675469c   1         1         1       33s
```

One Helm command created the Deployment, ReplicaSet, Pod and Service. All names start with the release name `my-app`, so the same chart can be installed twice under different names. `NOTES` are instructions written by the chart's author.

## Step 6 – Open the app

```bash
kubectl -n helm-lab port-forward deploy/my-app-podinfo 8080:9898 &
curl -s http://127.0.0.1:8080 | head -12
```

Expected: JSON with `"version": "6.12.0"` and the default `"message"`.

## Step 7 – Change one setting with `--set`

```bash
helm upgrade my-app podinfo/podinfo --version 6.12.0 -n helm-lab --set replicaCount=2
```

```text
Release "my-app" has been upgraded. Happy Helming!
NAME: my-app
LAST DEPLOYED: Thu Oct 15 19:40:22 2026
NAMESPACE: helm-lab
STATUS: deployed
REVISION: 2
```

```bash
kubectl get all -n helm-lab
```

```text
NAME                                 READY   STATUS    RESTARTS   AGE
pod/my-app-podinfo-ff675469c-hkqfs   1/1     Running   0          34m
pod/my-app-podinfo-ff675469c-xg6ld   1/1     Running   0          13s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-app-podinfo   2/2     2            2           34m
```

Result: 2 pods and **REVISION 2**. The existing pod kept running; Helm only added a second one.

## Step 8 – Change settings with a values file

Uses [`examples/custom-values.yaml`](examples/custom-values.yaml):

```yaml
replicaCount: 3
ui:
  message: "Hello from Abdul's Helm lab"
```

```bash
helm upgrade my-app podinfo/podinfo --version 6.12.0 -n helm-lab -f examples/custom-values.yaml
kubectl get pods -n helm-lab
curl -s localhost:8080 | grep message
helm get values my-app -n helm-lab
```

Expected: 3 pods, the custom message, and **REVISION 3**. If `curl` fails, restart the port-forward (`kill %1`, then run it again): the pod behind it may have been replaced.

## Step 9 – Upgrade to a newer chart version

```bash
helm upgrade my-app podinfo/podinfo --version 6.15.0 -n helm-lab -f examples/custom-values.yaml
helm list -n helm-lab
curl -s localhost:8080 | grep version
```

Expected: chart `podinfo-6.15.0`, app version `6.15.0`, **REVISION 4**.

## Step 10 – History and rollback

```bash
helm history my-app -n helm-lab
helm rollback my-app 2 -n helm-lab
kubectl get pods -n helm-lab
helm history my-app -n helm-lab
```

Expected: back to 2 pods on version 6.12.0. History shows a **new revision 5** described as "Rollback to 2". A rollback never deletes history; it adds a revision.

## Step 11 – Uninstall

```bash
kill %1                              # stop the port-forward
helm uninstall my-app -n helm-lab
kubectl get all -n helm-lab
kubectl delete ns helm-lab
```

Expected: every resource the chart created is removed with one command.

## What each step proves

| Concept | Lab step |
| --- | --- |
| Repository (like apt / dnf repo) | 2, 3 |
| Chart version vs app version | 3, 5, 9 |
| Values / configuration | 4, 7, 8 |
| Release and revisions | 5, 7, 8, 9, 10 |
| Rollback | 10 |
| Clean uninstall | 11 |
