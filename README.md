# Argo CD on Docker Desktop (kind) – Windows Setup

A quick guide to installing [Argo CD](https://argo-cd.readthedocs.io/) on a local Kubernetes cluster created by Docker Desktop's **kind** provisioner on Windows.

## Prerequisites

- Windows 10/11 with [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- `kubectl` (bundled with Docker Desktop)
- PowerShell

## 1. Enable Kubernetes in Docker Desktop

1. Open **Docker Desktop → Settings → General** and make sure **Use containerd for pulling and storing images** is enabled (required by kind).
2. Go to **Settings → Kubernetes**, enable Kubernetes, and select **kind** as the cluster provisioning method.
3. Choose the Kubernetes version and number of nodes, then click **Apply & restart**.

Verify the cluster is up:

```powershell
kubectl get nodes
```

## 2. Install Argo CD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all pods are `Running`:

```powershell
kubectl get pods -n argocd -w
```

## 3. Expose the Argo CD Server

Change the `argocd-server` service type to `LoadBalancer`:

```powershell
kubectl patch svc argocd-server -n argocd -p '{\"spec\":{\"type\":\"LoadBalancer\"}}'
```

> The inner quotes are escaped with `\` because Windows PowerShell 5.1 strips them otherwise (see [Troubleshooting](#troubleshooting)).

Check the assigned external IP:

```powershell
kubectl get svc argocd-server -n argocd
```

> With kind, the external IP (e.g. `172.18.0.6`) lives on Docker's internal network and is usually not reachable from Windows, so use port-forwarding in the next step.

## 4. Access the Argo CD UI

Start a port-forward and keep this terminal open:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open **https://localhost:8080** in your browser. Argo CD uses a self-signed certificate, so accept the browser warning (**Advanced → Proceed to localhost**).

## 5. Log In

Username: `admin`

Retrieve the initial password (in a second PowerShell window):

```powershell
$p = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

After logging in, change the password under **User Info → Update Password**, then delete the initial secret:

```powershell
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## Troubleshooting

### `invalid character 's' looking for beginning of object key string`

Windows PowerShell 5.1 strips double quotes from JSON arguments passed to native commands like `kubectl`. Escape the inner quotes:

```powershell
kubectl patch svc argocd-server -n argocd -p '{\"spec\":{\"type\":\"LoadBalancer\"}}'
```

Or use YAML syntax, which needs no quotes:

```powershell
kubectl patch svc argocd-server -n argocd -p 'spec: {type: LoadBalancer}'
```

PowerShell 7.3+ does not have this issue.

## Cleanup

```powershell
kubectl delete namespace argocd
```
