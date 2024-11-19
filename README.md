# Using K8ssandra to run Cassandra 5

## Add the K8ssandra helm repo
helm repo add k8ssandra https://helm.k8ssandra.io/stable
helm repo update

## Clone the K8ssandra operator
git clone https://github.com/k8ssandra/k8ssandra-operator.git
cd k8ssandra-operator

## Setup a local kind cluster with 1 control plane and 4 worker nodes
scripts/setup-kind-multicluster.sh --clusters 1 --kind-worker-nodes 4

## List the nodes in the cluster
kubectl get nodes

## Add the cert-manager helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

## Use the control plane context
kubectl config use-context kind-k8ssandra-0

## Install cert-manager
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true

## List the pods in the cert-manager namespace
kubectl get pods --namespace cert-manager

## Deploy the K8ssandra operator in a dedicated namespace
helm install k8ssandra-operator k8ssandra/k8ssandra-operator -n k8ssandra-operator --create-namespace

## Verify the operator pods are running
kubectl get pods -n k8ssandra-operator

## Deploy a K8ssandra cluster
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

## Create the K8ssandra cluster
kubectl apply -n k8ssandra-operator -f ../k8c1.yml

## Verify pod deployment
kubectl get pods -n k8ssandra-operator

## Verify K8ssandraCluster deployment
kubectl get k8cs -n k8ssandra-operator

## Describe the K8ssandraCluster
kubectl describe k8cs demo -n k8ssandra-operator

## Extract credentials from the K8ssandraCluster
CASS_USERNAME=$(kubectl get secret demo-superuser -n k8ssandra-operator -o=jsonpath='{.data.username}' | base64 --decode)
echo $CASS_USERNAME
-> demo-superuser
CASS_PASSWORD=$(kubectl get secret demo-superuser -n k8ssandra-operator -o=jsonpath='{.data.password}' | base64 --decode)
echo $CASS_PASSWORD
-> kEFiYwQ-AbWpEumJ5A4H

## Verify cluster status
kubectl exec -it demo-dc1-default-sts-0 -n k8ssandra-operator -c cassandra -- nodetool -u $CASS_USERNAME -pw $CASS_PASSWORD status

## Open a shell in the cassandra pod
kubectl exec -it demo-dc1-default-sts-0 -n k8ssandra-operator -- /bin/bash
nodetool -u demo-superuser -pw kEFiYwQ-AbWpEumJ5A4H status
cqlsh -u demo-superuser -p kEFiYwQ-AbWpEumJ5A4H 10.244.1.5

Connected to demo at 10.244.1.5:9042
[cqlsh 6.2.0 | Cassandra 5.0.2 | CQL spec 3.4.7 | Native protocol v5]
Use HELP for help.

# Prerequisites

## Install kind as a local kubernetes cluster
brew install kind

## Install kubectx (Simplifies the management of Kubernetes contexts and namespaces)
kubectl krew install ctx
kubectl krew install ns
Use as: kubectl ctx

### Optionally install the bash completion
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

## Install Yq (YAML processing tool)
brew install yq

## Install gnu-getopt (Required for kubectl-krew)
brew install gnu-getopt
echo 'export PATH="/opt/homebrew/opt/gnu-getopt/bin:$PATH"' >> /Users/michel.deru/.zshrc
source ~/.zshrc