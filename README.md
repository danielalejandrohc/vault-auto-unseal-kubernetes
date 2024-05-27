### unseal vault in Kubernetes

Welcome! I'm glad you're here, as visits to this page might be minimal since I no longer maintain my personal blog where I posted a few how-to articles on topics that I found interesting.

Recently, I've been curious about HashiCorp Vault. Although I used it a long time ago, I remember it being a powerful, useful, and secure tool for storing secrets.

The free version of Vault requires "unsealing," which means you have to execute some commands with keys revealed after installing Vault. In this guide, I'd like to share a simple yet functional medium-scale implementation of Vault. I've found that many customers see the need to unseal Vault as a hassle rather than a benefit. While it's designed to enhance security, many of us prefer an implementation that can automatically recover if the Pod dies for any reason, such as a Node upgrade, Vault upgrade, etc.

So, let's get started.

Note: This implementation is for Vault running in a Kubernetes cluster. If you need to do this in a simpler architecture based on containers, you can adapt it, for example, using plain Docker, Docker Compose, etc.

### Step 1: Create Namespace

```
kubectl create ns vault
```

### Step 2: Create a unseal placeholder script:

This Secret is our "super trick" this Secret it is an Script that will run when PostHook event in the Pod, it is a Script that it halts a bit bringing some seconds to Vault to boot, and will unseal it. The first time you won't know It secrets so after the installation you will have to come again and update it properly.

Content of unseal.sh
```
wait 10
# After the first deployment this file will look different, for example with below unseal commands
# vault operator unseal x
# vault operator unseal x
# vault operator unseal x
```

### Step 3: Generata base64 String from the Script

Generate the content of the previous file as a base64 encoded String


```
cat unseal.sh | base64
```

Store this output and keep it in your clipboard.

### Step 4: Generate Secret 

Create the Secret of Kubernetes


```
vim unseal.yaml
```

Content:

```
# unseal.yaml
apiVersion: v1
kind: Secret
metadata:
  name: unseal-script
  namespace: vault
type: Opaque
data:
  unseal.sh: #CONTENT from Previous step#
```

```
kubectl apply -f unseal.yaml
```

### Step 5: Install Vault using Helm

Please, prior running this, check values.yaml you have to choose the proper Storage Class for your cluster.

```
# Install it from Helm
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --values values.yaml -n vault
```

### Step 6: Get the keys!

If you are not new to Vault you in the output commands will be the unseal command:

```
kubectl logs pod/vault-0 -n vault
```

### Step 7: Update the Secret

In here you have to basically go to Step 3 and Step 4 and update the unseal.sh script properly, with the new keys, for example It would look like this:

```
echo Unsealing ...
sleep 10
vault operator unseal abccc205258dba9ed2537b08456
vault operator unseal abccc205258dba9ed2537b08321
vault operator unseal abccc205258dba9ed2537b08123
```

### Step 8: Delete vault Pod

Since, the Secret has changed you need to redeploy the Pod, in this implementation what you get it is a Pod named "vault-0" (Behavior at May 2024), you need to delete it.

```
kubectl delete pod vault-0
```

### Step 9: Enjoy and secure!

That's it, now you can start customizing Vault as you wish, Vault will auto unseal after every deploy.