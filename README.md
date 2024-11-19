# Using K8ssandra to run Cassandra 5
This repository contains a runbook to set up a kind K8s cluster and deploy Cassandra 5 using K8ssandra and cass-operator.
## Add the K8ssandra helm repo
```bash
helm repo add k8ssandra https://helm.k8ssandra.io/stable
helm repo update
```

## Clone the K8ssandra operator
```bash
git clone https://github.com/k8ssandra/k8ssandra-operator.git
cd k8ssandra-operator
```

## Setup a local kind cluster with 1 control plane and 4 worker nodes
```bash
scripts/setup-kind-multicluster.sh --clusters 1 --kind-worker-nodes 4
```

### List the nodes in the cluster
```bash
kubectl get nodes
```

Output:
```text
NAME                        STATUS   ROLES                  AGE   VERSION
k8ssandra-0-control-plane   Ready    control-plane,master   80s   v1.22.4
k8ssandra-0-worker          Ready    <none>                 42s   v1.22.4
k8ssandra-0-worker2         Ready    <none>                 42s   v1.22.4
k8ssandra-0-worker3         Ready    <none>                 42s   v1.22.4
k8ssandra-0-worker4         Ready    <none>                 42s   v1.22.4
```

## Add the cert-manager helm repo
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

## Use the control plane context
```bash
kubectl config use-context kind-k8ssandra-0
```

## Install cert-manager
```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
```

### List the pods in the cert-manager namespace
```bash
kubectl get pods --namespace cert-manager
```

Output:
```text
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5cc5dcf776-bmz4z              1/1     Running   0          31m
cert-manager-cainjector-7dcc56c84f-dm8hf   1/1     Running   0          31m
cert-manager-webhook-5dcf8fd964-kxfbl      1/1     Running   0          31m
```

## Deploy the K8ssandra operator in a dedicated namespace
```bash
helm install k8ssandra-operator k8ssandra/k8ssandra-operator -n k8ssandra-operator --create-namespace
```

## Verify the operator pods are running
```bash
kubectl get pods -n k8ssandra-operator
```

Output:
```text
NAME                                                READY   STATUS    RESTARTS   AGE
k8ssandra-operator-7f76579f94-7s2tw                 1/1     Running   0          60s
k8ssandra-operator-cass-operator-794f65d9f4-j9lm5   1/1     Running   0          60s
```

## Deploy a K8ssandra cluster 🤩
We use a custom values.yaml file to deploy a 3 node cluster in one datacenter.
`k8c1.yml`:
```yaml
apiVersion: k8ssandra.io/v1alpha1
kind: K8ssandraCluster
metadata:
  name: demo
spec:
  cassandra:
    serverVersion: "5.0.2"
    datacenters:
      - metadata:
          name: dc1
        size: 3
        storageConfig:
          cassandraDataVolumeClaimSpec:
            storageClassName: standard
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 5Gi
        config:
          jvmOptions:
            heapSize: 512M
        stargate:
          size: 1
          heapSize: 256M
```

### Create the K8ssandra cluster
```bash
kubectl apply -n k8ssandra-operator -f ../k8c1.yml
```

### Verify pod deployment
```bash
kubectl get pods -n k8ssandra-operator
```

Output:
```text
NAME                                                   READY   STATUS             RESTARTS       AGE
demo-dc1-default-stargate-deployment-b6f6969d4-kmthp   0/1     CrashLoopBackOff   11 (33s ago)   23m
demo-dc1-default-sts-0                                 2/2     Running            0              27m
demo-dc1-default-sts-1                                 2/2     Running            0              27m
demo-dc1-default-sts-2                                 2/2     Running            0              27m
k8ssandra-operator-6fc9c77c68-j6m8g                    1/1     Running            0              31m
k8ssandra-operator-cass-operator-d54556f7c-lg6gs       1/1     Running            0              31m
```

### Verify K8ssandraCluster deployment
```bash
kubectl get k8cs -n k8ssandra-operator
```

Output:
```text
NAME   ERROR
demo   None
```

### Describe the K8ssandraCluster
```bash
kubectl describe k8cs demo -n k8ssandra-operator
```

### Extract credentials from the K8ssandraCluster
```bash
CASS_USERNAME=$(kubectl get secret demo-superuser -n k8ssandra-operator -o=jsonpath='{.data.username}' | base64 --decode)
echo $CASS_USERNAME
```
Output: demo-superuser
```bash
CASS_PASSWORD=$(kubectl get secret demo-superuser -n k8ssandra-operator -o=jsonpath='{.data.password}' | base64 --decode)
echo $CASS_PASSWORD
```
Output: kEFiYwQ-AbWpEumJ5A4H

### Verify cluster status
```bash
kubectl exec -it demo-dc1-default-sts-0 -n k8ssandra-operator -c cassandra -- nodetool -u $CASS_USERNAME -pw $CASS_PASSWORD status
```

Output:
```text
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load        Tokens  Owns (effective)  Host ID                               Rack   
UN  10.244.1.5  150.03 KiB  16      100.0%            ed9795cd-665a-41b3-bdd0-85020e2ec88e  default
UN  10.244.2.4  114.67 KiB  16      100.0%            2e07f986-2cf0-4b14-a764-d7dc3cdc2610  default
UN  10.244.3.4  114.74 KiB  16      100.0%            622eab4a-6358-4d29-8e7a-1fc397bd3ceb  default
```

### Open a shell in the cassandra pod and run cqlsh
```bash
kubectl exec -it demo-dc1-default-sts-0 -n k8ssandra-operator -- /bin/bash
nodetool -u demo-superuser -pw kEFiYwQ-AbWpEumJ5A4H status
cqlsh -u demo-superuser -p kEFiYwQ-AbWpEumJ5A4H 10.244.1.5
```

Output:
```text
Connected to demo at 10.244.1.5:9042
[cqlsh 6.2.0 | Cassandra 5.0.2 | CQL spec 3.4.7 | Native protocol v5]
Use HELP for help.
```

# Prerequisites

## Install kind as a local kubernetes cluster
```bash
brew install kind
```

## Install kubectx (Simplifies the management of Kubernetes contexts and namespaces)
```bash
kubectl krew install ctx
kubectl krew install ns
```
Use as: kubectl ctx

### Optionally install the bash completion
```bash
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

## Install Yq (YAML processing tool)
```bash
brew install yq
```

## Install gnu-getopt (Required for kubectl-krew)
```bash
brew install gnu-getopt
echo 'export PATH="/opt/homebrew/opt/gnu-getopt/bin:$PATH"' >> /Users/michel.deru/.zshrc
source ~/.zshrc
```
