# Istio Multi-cluster on separate networks

Reference: https://istio.io/latest/docs/setup/install/multicluster/primary-remote_multi-network/

Not shown: I created two VPCs, one named network1 in the us-central1 region, and the other network2 in us-east1.

## Create the clusters

gcloud container clusters create central-cluster \
  --cluster-version latest \
  --machine-type "e2-standard-2" \
  --num-nodes "3" \
  --network "network1" \
  --subnetwork "network1-subnet" \
  --region "us-central1"


gcloud container clusters create east-cluster \
  --cluster-version latest \
  --machine-type "e2-standard-2" \
  --num-nodes "3" \
  --network "network2" \
  --subnetwork "network2-subnet" \
  --region "us-east1"

## Create common root of trust

Reference: https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/

### Create the certs and keys

Make the root ca first.

Then:

make -f ../tools/certs/Makefile.selfsigned.mk central-cluster-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk east-cluster-cacerts

## Create the secrets

Create the `istio-system` namespace and create secret in each kube context.

kubectx gke_eitan-tetrate_us-east1_east-cluster
pushd east-cluster
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=ca-cert.pem \
      --from-file=ca-key.pem \
      --from-file=root-cert.pem \
      --from-file=cert-chain.pem
popd

kubectx gke_eitan-tetrate_us-central1-a_central-cluster
pushd central-cluster
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=ca-cert.pem \
      --from-file=ca-key.pem \
      --from-file=root-cert.pem \
      --from-file=cert-chain.pem
popd

3. Configure your context env vars:

export CTX_CLUSTER1=gke_eitan-tetrate_us-central1-a_central-cluster
export CTX_CLUSTER2=gke_eitan-tetrate_us-east1_east-cluster
