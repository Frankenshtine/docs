---
description: Deploying a THORNode and its associated services.
---

# Deploying

## **Deploy THORNode services**

Now you have a Kubernetes cluster ready to use, you can install the THORNode services.

{% hint style="info" %}
Helm charts are the defacto and currently easiest and simple way to package and deploy Kubernetes application. The team created different Helm charts to help to deploy all the necessary services. Please retrieve the source files from the Git repository here to follow the instructions below:

\*\*\*\*[ **https://gitlab.com/thorchain/devops/node-launcher**](https://gitlab.com/thorchain/devops/node-launcher)\*\*\*\*
{% endhint %}

### Requirements

* Running Kubernetes cluster
* Kubectl configured, ready and connected to running cluster

{% hint style="info" %}
If you came here from the Setup page, you are already good to go.
{% endhint %}

## Steps

Clone the `node-launcher` repo. All commands in this section are to be run inside of this repo.

```text
git clone https://gitlab.com/thorchain/devops/node-launcher
cd node-launcher
git checkout multichain
```

### Install Helm 3

Install Helm 3 if not already available on your current machine:

```text
make helm
```

### Tools

{% tabs %}
{% tab title="Deploy" %}
To deploy all tools, metrics, logs management, Kubernetes Dashboard, run the command below.

```text
make tools
```
{% endtab %}

{% tab title="Destroy" %}
To destroy all those resources run the command below.

```text
make destroy-tools
```
{% endtab %}
{% endtabs %}

If you are successful, you will see the following message:

![](../.gitbook/assets/image%20%2823%29.png)

If there are any errors, they are typically fixed by running the command again.

## Deploy THORNode

It is important to deploy the tools first before deploying the THORNode services as some services will have metrics configuration that would fail and stop the THORNode deployment.

You have multiple commands available to deploy different configurations of THORNode. You can deploy testnet or chaosnet/mainnet. The commands deploy the umbrella chart `thornode-stack` in the background in the Kubernetes namespace `thornode` \(or `thornode-testnet` for testnet\) by default.

```text
make install
```

{% hint style="info" %}
If you are intending to run all chain clients, bond in & earn rewards, you want to choose "Validator".
{% endhint %}

{% hint style="info" %}
Deploying a THORNode will take 1 day for every 3 months of ledger history, since it will validate every block. THORNodes are "full nodes", not light clients.
{% endhint %}

If successful, you will see the following:

![](../.gitbook/assets/image%20%2819%29.png)

You are now ready to join the network:

{% page-ref page="joining.md" %}

### Debugging

{% hint style="info" %}
Set `thornode` to be your default namespace so you don't need to type `-n thornode` each time:

`kubectl config set-context --current --namespace=thornode`
{% endhint %}

Use the following useful commands to view and debug accordingly. You should see everything running and active. Logs can be retrieved to find errors:

```text
kubectl get pods -n thornode
kubectl get pods --all-namespaces
kubectl logs -f <pod> -n thornode
```

Kubernetes should automatically restart any service, but you can force a restart by running:

```text
kubectl delete pod <pod> -n thornode
```

{% hint style="warning" %}
Note, to expedite syncing external chains, it is feasible to continually delete the pod that has the slow-syncing chain daemon \(eg, binance-daemon-xxx\).

Killing it will automatically restart it with free resources and syncing is notably faster. You can check sync status by viewing logs for the client to find the synced chain tip and comparing it with the real-world blockheight, \("xxx" is your unique ID\):

```text
kubectl logs -f binance-daemon-xxx -n thornode
```
{% endhint %}

{% hint style="info" %}
Get real-world blockheights on the external blockchain explorers, eg:  
[https://testnet-explorer.binance.org/](https://testnet-explorer.binance.org/)

[https://explorer.binance.org/](https://explorer.binance.org/)
{% endhint %}

## Restoring node after disaster

There is possible disaster case scenario for:
* Accidental node destruction
* Hardware or service failure / corruption
* Cloud provider account blocking
* etc
- events preventing active node work and requiring to rebuild same node ASAP, possibly on different cloud provider.

### Prepare yourself to disaster

To rebuild same node, you need to restore k8s secret, containing node vault mnemonic. Simplest way is to export this secret to file and store it safe.
```text
kubectl get secret thornode-mnemonic -n thornode-testnet -o yaml > mnemonic_secret.yaml
```

If you didn't do that while node is healthy and reachable, but have dumped mnemonic (via `make mnenonic` command), you can make yaml file manually. Use this as a sample:
```text
apiVersion: v1
data:
  mnemonic: Z2F1Z2UgYnJpY2sgb3JpZW50IHJhcGlkIHRyYWRlIHNoaWVsZCBnaXJsIHVnbHkgZ2VudGxlIGRhcmluZyBzcG9pbCBjdXRlIGRlc3Ryb3kgaW5jb21lIGVtcGxveSBjZWlsaW5nIHJlZ3JldCBpbmZhbnQgZmlndXJlIHVuYXdhcmUgdW5pZm9ybSBraXRjaGVuIHN3YW1wIG51dA==
kind: Secret
metadata:
  creationTimestamp: "2021-11-15T07:57:02Z"
  name: thornode-mnemonic
  namespace: thornode
  resourceVersion: "3920"
  uid: 6cfa8a22-a8b5-4446-9d7c-8b97168296da
type: Opaque
```
Fields creationTimestamp, resourceVersion and uid can have any valid value, but mnemonic is a base64-encoded string your node's mnemonic. You can generate it by command
```text
echo -n "gauge brick orient rapid trade shield girl ugly gentle daring spoil cute destroy income employ ceiling regret infant figure unaware uniform kitchen swamp nut" | base64 -w0
```
Of course, use your mnemonic instead of example. Paste encoded string in a file, and you ready to restore your node. Remember to save this file safe, as it containing mnemonic for your node's vault!

### Restoring node using a mnemonic secret

After spinning a new cluster, first create some k8s stuff. You need to create namespace where will be saved secret:
```text
kubectl create namespace thornode
```

Now use you secret file to create s k8s secret:
```text
kubectl apply -f mnemonic_secret.yaml
```
Now, go to normal validator install procedure and wait for a sync. After your new thornode catch a sync to the tip, last thing will be broadcasting new IP address to the network, run `make set-ip-address` from node-launcher repo folder.

## CHART SUMMARY

#### THORNode full stack umbrella chart

* **thornode**: Umbrella chart packaging all services needed to run a fullnode or validator THORNode.

This should be the only chart used to run THORNode stack unless you know what you are doing and want to run each chart separately \(not recommended\).

#### THORNode services:

* **thor-daemon**: THORNode daemon
* **thor-api**: THORNode API
* **thor-gateway**: THORNode gateway proxy to get a single IP address for multiple deployments
* **bifrost**: Bifrost service
* **midgard**: Midgard API service

#### External services:

* **binance-daemon**: Binance fullnode daemon
* **bitcoin-daemon**: Bitcoin fullnode daemon
* **ethereum-daemon**: Ethereum fullnode daemon
* **chain-daemon**: as required for supported chains

#### Tools

* **elastic**: ELK stack, deperecated. Use elastic-operator chart
* **elastic-operator**: ELK stack using operator for logs management
* **prometheus**: Prometheus stack for metrics
* **loki**: Loki stack for logs
* **kubernetes-dashboard**: Kubernetes dashboard

