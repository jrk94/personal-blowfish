---
title: "MES on a Local Kubernetes Cluster in WSL: Four Traps Between a Clean Apply and a Working URL"
description: "Installing MES with DevOps Center in manual mode on a single-node Kubernetes cluster in WSL. A bare cluster has no default storage class and no LoadBalancer implementation, and on any cluster the Traefik CRDs and the name resolution are your job. The YAML applies without errors and the environment does not open."
summary: "The eight-step DevOps Center flow on a bare single-node cluster in WSL, and four problems that produce no clear error: no storage class, Traefik CRDs missing or the wrong version, a LoadBalancer stuck on pending, and a name Windows cannot resolve. Each comes with symptom, reason and fix."
categories: ["MES"]
tags: ["DevOps Center", "Kubernetes", "WSL", "Traefik", "MetalLB", "Deployment"]
date: 2026-11-20
showInRSS: true
draft: false
authors:
  - Roque
---
A local Kubernetes cluster will accept every YAML file DevOps Center generates. Every `kubectl apply` returns `created`, and you can still end up with an environment you cannot open.

A managed cluster provides a default storage class and a LoadBalancer implementation. A bare single-node cluster in WSL has neither, so the volumes stay unbound and the Traefik service stays on `<pending>`. Those two are visible, but easy to misread. The other two are on you on any cluster: the Traefik CRDs that match the agent's Traefik, and a name that Windows can resolve. The CRD one gives no obvious error at all.

## Before You Start

This is the second post of two. The [first post](https://j-roque.com/posts/20261119-devops-center-install/) explains the concepts: what DevOps Center, an infrastructure, the infrastructure agent and a customer environment are, the installation paths and the backups. This post covers the hands-on part, so I assume you know what an agent and an environment are.

The setup this post follows is a colleague's single-node Kubernetes cluster inside WSL (Ubuntu) on a laptop, not a production cluster. The deployment target in DevOps Center is Kubernetes, with the deployment mode set to **Manual**. A single worker node is enough to learn the flow. A production cluster, such as a Red Hat OpenShift one (a Kubernetes distribution), has several.

The post assumes a bare cluster, the kind you get from a kubeadm-style install. Distributions such as k3s, minikube, kind and Docker Desktop ship some of these defaults (storage, a load balancer or an ingress), so the traps change there. Check what yours ships before you install anything. In that setup the cluster commands ran as root inside WSL.

Every command block below is a sketch to adapt, not a transcript. I adapted them from the official Kubernetes, Traefik, MetalLB and local-path-provisioner documentation, and angle brackets mark the placeholders you replace. Anything I could not confirm is collected in [What I Did Not Verify](#what-i-did-not-verify).

## The Eight-Step Flow

The flow has eight steps and produces two ZIP files:

![Eight-step flow: DevOps Center, Create Infrastructure, Create Infrastructure Agent producing an agent ZIP of YAML, Deploy Agent to Cluster, Agent Connected, Create Customer Environment producing an environment ZIP, Deploy MES, and Validate pods and namespace](img/eight-step-flow.png "The eight steps. DevOps Center generates two ZIP files, the agent YAML and the environment YAML. In manual mode you apply both yourself.")

Steps 1 to 3 are in the portal: open DevOps Center, create the infrastructure and create the infrastructure agent, which produces the agent ZIP. Step 4 is yours: deploy the agent to the cluster. Step 5 is the agent turning green. Steps 6 and 7 create the environment and deploy it. Step 8 is you looking at pods and namespaces.

The diagram shows the portal and the cluster side by side. It does not show what the cluster must have before step 4, which is where the traps are.

### The Order I Would Use

On the cluster, this order avoids the traps:

1. The cluster itself, healthy.
2. A default storage class.
3. The Traefik CRDs that match the agent.
4. Deploy the agent.
5. Create and deploy the environment.
6. A LoadBalancer implementation and an external IP for the Traefik service.
7. The hosts file entry on Windows.

Portal steps 1 to 3 come first, because the agent version decides which CRDs you need.

## Manual Versus Remote

In **remote** mode, the cluster pulls everything through the agent and the portal. In **manual** mode, you download a ZIP and apply the YAML files yourself.

My understanding is that this is the only difference between the two modes. Remote needs a connected agent to deploy anything. Manual needs one too, as you will see in step 7, because the Environment Manager still downloads packages through it.

I would use manual mode for a local cluster for a practical reason: you see every object that lands in the cluster.

## Steps 1 to 3: DevOps Center, Infrastructure and Agent

In DevOps Center, create the infrastructure with target Kubernetes and deployment mode Manual. Then create the infrastructure agent and download the agent ZIP. Extract it into a folder that WSL can see.

The infrastructure agent is the piece that connects your cluster to the portal. I covered its three components in the [first post](https://j-roque.com/posts/20261119-devops-center-install/). The older [Edge Deploy post](https://j-roque.com/posts/20250721-edge-deploy/) shows the same agent enrolling a cluster that sits next to the shop floor.

## Prepare the Cluster

Create the single-node cluster in WSL and check that it is healthy before you apply anything from DevOps Center. This post does not depend on a distribution.

Create a user or service account with cluster admin permissions, then check the nodes and the system pods:

```bash
kubectl get nodes
kubectl get pods -A
```

If the system pods are not running, nothing after this works, and you will blame the wrong layer. If you want a UI, the [Kubernetes Dashboard documentation](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) states that the Dashboard is deprecated and recommends Headlamp for new installations.

### Trap 1: No Storage Class

**Symptom:** the environment YAML applies without an error, and the environment never comes up. The services that need volumes are the ones waiting.

**Reason:** every service needs volumes, for logs and shared data. The volumes are requested through persistent volume claims, and a claim needs a storage class to be provisioned dynamically. Managed clusters usually ship a default one, and a bare cluster does not. The [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/) says a claim with no `storageClassName` is created as written, with no class set until a default appears, and the [persistent volumes page](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) says that claims stay unbound until a matching volume exists.

The apply succeeds because it only creates the claims, and they stay unbound until a volume is provisioned. Three commands show it:

```bash
kubectl get storageclass
kubectl get pvc -A
kubectl describe pod <pod> -n <environment-namespace>
```

Look for no storage class marked as default, claims in `Pending`, and pod events that mention unbound claims.

**Fix:** add a storage class and make it the default. For a single node, the simplest option is the Rancher [local-path-provisioner](https://github.com/rancher/local-path-provisioner). It creates `hostPath` or `local` volumes on the node, installs into the `local-path-storage` namespace and registers a storage class called `local-path`. The project page does not say whether that class is the default, so mark it yourself.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl annotate storageclass local-path storageclass.kubernetes.io/is-default-class=true
```

The alternative, and what that setup used, is an NFS provisioner in its own namespace. It requires an NFS server that the node can reach, in that setup on `localhost` exporting a folder in WSL that I had to create first, with permissions that let the services read and write. Many WSL kernels lack NFS server support, so check before you choose it. Kubernetes has no built-in NFS provisioner, and the storage classes page says an external one can be used. With NFS, a provisioner pod binds the claims dynamically and you see folders appear in the exported path as the environment deploys.

Whatever you pick, it must be the default, and there is a catch in how defaults behave:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <your-class>
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: <the-provisioner-name>
```

Per the Kubernetes documentation, if a default class appears later, existing claims with an unset `storageClassName` are updated to use it. If the environment YAML sets a class name explicitly, a default will not help, and you need a class with that name. Which case applies depends on the YAML, and I did not check what it sets. That is why the storage class goes in before the environment, not after.

### Trap 2: Traefik CRDs Missing or the Wrong Version

This is the one I would check first, on any cluster. It is a common cause of "application not available" after an install or an upgrade, and it is the one that gives no obvious error.

**Symptom:** the install finishes, the pods run, and the URL says the application is not available. The UI and the Environment Manager are both unreachable. Nothing in the portal tells you why.

**Reason:** the agent deploys Traefik, and the ingress routes of the environment are Traefik custom resources. If the Traefik CRDs are missing, or older than the version the agent's Traefik expects, Traefik cannot understand the routes that DevOps Center ships. There is no route, so there is nothing to open.

In that setup, the agent for 11.2 needed the CRDs for Traefik v3.6. Read the Traefik image tag from your agent package, not from this post.

To confirm the symptom, list the resources and read the Traefik logs once the agent is running:

```bash
kubectl get crd | grep traefik
kubectl get ingressroute -A
kubectl -n <agent-namespace> logs deployment/<traefik-deployment>
```

If the CRDs are missing, `kubectl` should not know the `ingressroute` resource type at all. If they are present but too old, look in the Traefik log for errors from its Kubernetes provider. I did not capture the exact messages.

**Fix:** install the CRDs before you deploy the agent. The [Traefik documentation](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-crd/) says to apply the definitions from the Traefik repository, with the version in the URL. The page I opened was for v3.7, and the same path exists for v3.6.

```bash
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.6/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
```

If the agent is already running, restart only the Traefik deployment so it picks the definitions up:

```bash
kubectl -n <agent-namespace> rollout restart deployment/<traefik-deployment>
```

Do not restart every deployment in the namespace. That restarts the Edge Forward Proxy too, and the agent goes red again until it reconnects.

The same Traefik page lists an RBAC file next to the definitions, documented for installs that do not use the Helm chart. I did not check whether the agent already ships that RBAC, so look at what your agent ZIP contains.

> If the environment is up and the URL says unavailable, check the Traefik CRDs before you touch anything else.

## Step 4: Deploy the Agent

Apply the agent YAML files with `kubectl apply`, pointing at the folder where you extracted the ZIP. The packages also ship PowerShell scripts, so install `pwsh` in WSL first and run them with it.

```bash
kubectl apply -f <agent-folder>
```

The result is a new namespace named after the agent, with three pods: the Edge Forward Proxy, the Edge Squid Proxy and Traefik. Check them:

```bash
kubectl get pods -n <agent-namespace>
```

## Step 5: Agent Connected

The agent shows **red** in DevOps Center when it is not connected and **green** when it is. It started red. A few minutes later, when the Edge Forward Proxy pod came up, it turned green.

That pod can take a few minutes to start, and red only means the agent is not connected yet, so look at the pod before you delete anything. If it stays red, check that pod first. The agent connects outbound to the portal, so it should need outbound access and no inbound port.

Do not move to step 6 until it is green. Without a connected agent you cannot install anything.

## Step 6: Create the Environment

Create the customer environment in DevOps Center: a system name, a tenant name and the **public URL**. In that setup it is a `.local` name, and here I use the placeholder `mes.local`. It becomes the entry point that Traefik uses as the main URL, and it matters again in the fourth trap. Then download the environment ZIP.

## Step 7: Deploy MES

Apply the environment YAML files. They arrive as a stack file that a PowerShell script from the ZIP runs, so use `pwsh` for the script. The script's name and arguments are not given here, so read it before you run it.

```bash
pwsh ./<script-from-the-environment-zip>.ps1
kubectl apply -f <environment-folder>
```

Which of the two your package expects depends on its contents.

This was a clean installation: no database and no packages. The Environment Manager fetches the product baseline database and the packages for the MES version, 11.2.3 in that setup, and it downloads them through the agent. That is why the agent has to be connected. It restores the baseline and installs. It takes some minutes.

Watch the provisioner create folders as claims bind, which shows the first trap is behind you. When the pods run, the last two traps are left.

### Trap 3: The LoadBalancer Stuck on Pending

**Symptom:** the pods are running and the environment is deployed. `kubectl get services` shows the Traefik service with `EXTERNAL-IP` as `<pending>`, and it stays that way.

**Reason:** a service of type `LoadBalancer` needs something to provide the load balancer. The [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/) says that Kubernetes does not offer a load balancing component directly: you provide one, or integrate the cluster with a cloud provider. Managed clusters do the second. A bare cluster does neither.

```bash
kubectl get services -n <agent-namespace>
```

**Fix:** install a bare-metal implementation. That setup used [MetalLB](https://metallb.io/installation/). The steps are install it, wait for its pods, create an address pool, create a layer 2 advertisement and give the Traefik service an address from the pool. The release in the URL is the one on the installation page when I read it, so check the current one. The page also lists an FRR-K8s manifest as the recommended mode. I show the native one as the simplest, and I have not compared the two.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.2/config/manifests/metallb-native.yaml
kubectl -n metallb-system get pods
```

The installation does not include a configuration, so nothing is assigned until you add one. The [configuration page](https://metallb.io/configuration/) shows both resources. The range here is a placeholder, and the next subsection explains how to choose a real one:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.30.98.240-172.30.98.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-l2
  namespace: metallb-system
```

An `L2Advertisement` without a pool selector applies to every pool. An early `connection refused` when applying these means the MetalLB pods are not ready yet, so wait and retry.

To pin the Traefik service to a known address, the [MetalLB usage page](https://metallb.io/usage/) documents an annotation:

```bash
kubectl -n <agent-namespace> annotate service <traefik-service> metallb.io/loadBalancerIPs=172.30.98.240
```

The service is owned by the agent YAML, so applying that YAML again can revert a hand-made annotation. A pool with a single address achieves the same result and survives a re-apply. Check the service again, and it shows an external IP.

#### Choosing the Pool in WSL

MetalLB in layer 2 mode answers ARP requests on the local network, so the pool must come from a subnet that Windows can reach. By default, [WSL uses NAT](https://learn.microsoft.com/windows/wsl/networking): the VM has its own virtual adapter and its own IP, which you read from Windows with `wsl hostname -I`. Mirrored networking mode needs Windows 11 22H2 or later.

To pick addresses, look at the subnet inside WSL and choose a few unused ones in it:

```bash
ip -4 addr show eth0
ping -c 1 <candidate-address>
```

The first command shows the VM address and prefix length, and the pool goes in that subnet, away from the VM's own address. A candidate that does not answer the ping is probably free. In NAT mode, the VM address can change after a restart, in my understanding, and that breaks both a fixed pool and the hosts file entry from the next trap. Check `wsl hostname -I` after a restart. I did not test the pool choice on every networking mode.

### Trap 4: The Name Windows Cannot Resolve

**Symptom:** the service has an external IP and the browser still says the site cannot be reached.

**Reason:** the environment answers on the public URL you typed in step 6, `mes.local` in this post. Nothing resolves that name to the Traefik IP.

**Fix:** add an entry to the Windows hosts file, `C:\Windows\System32\drivers\etc\hosts`, that maps the external IP of the Traefik service to the public URL. Open it in an elevated editor. The browser runs on Windows, so the entry goes in the Windows file, not the one inside WSL.

```text
172.30.98.240   mes.local
```

The address and name are placeholders. Use the IP from trap 3 and the exact URL you configured in the environment. The `.local` suffix is reserved for multicast DNS, so a suffix such as `.test` is the safer placeholder.

## Step 8: Validate

List the namespaces and the pods, then open the URL:

```bash
kubectl get namespaces
kubectl get pods -n <environment-namespace>
```

Open the public URL you configured, `mes.local` in this post. MES is running. The first login prompts for a new password for the initial user, as usual.

## An Open Question: Skipping the Agent's Traefik

Some customers do not want the agent's Traefik, because they already run their own ingress or because of security rules. There is an option in the DevOps Center deployment to skip Traefik. Those customers create an ingress that points straight at the Traefik inside the MES stack.

The open question: **which Traefik version should the CRDs match, the one in the stack or the one in the agent?**

An 11.2 agent runs Traefik v3.6, and a 10.2 stack ships an older Traefik, v2. The [Traefik v2 to v3 migration details](https://doc.traefik.io/traefik/migrate/v2-to-v3-details/) state that the `traefik.containo.us` CRD API group was removed in v3 in favor of `traefik.io`, so the two generations are not interchangeable. I could not determine which set an ingress pointing at the stack's Traefik needs. If you run this setup, test it on a throwaway cluster first.

## What I Did Not Verify

- That manual and remote mode differ only in who applies the YAML.
- The option to skip the agent's Traefik. The DevOps Center pages sit behind a login, so I could not check it in the public help.
- That an 11.2 agent needs the Traefik v3.6 CRDs and that a 10.2 stack ships Traefik v2. Both come from that setup, not from public documentation.
- Whether the agent package already ships the Traefik RBAC.
- The exact output of `kubectl get pvc`, `kubectl describe pod`, `kubectl get ingressroute` and the Traefik logs in the failing cases. I describe what I expect, not captured output.
- That local-path-provisioner works with the MES environment volumes. That setup used NFS.
- That the WSL kernel you run has NFS server support.
- That the WSL VM address changes after a restart, and that the pool choice works on every networking mode.
- That a hand-made annotation on the Traefik service is reverted when the agent YAML is applied again.
- That `.local` causes trouble here. It is reserved for multicast DNS, but I did not test it.
- MetalLB's native mode against its recommended FRR-K8s mode, and the pinned release.

## Final Thoughts

A bare cluster leaves two jobs that a managed one does for you, a default storage class and a LoadBalancer implementation. Two more are yours on any cluster: the Traefik CRDs that match the agent, and a name Windows can resolve.

If the environment on a local cluster does not open, check the storage class, the Traefik CRDs, the load balancer and the name resolution, in that order.
