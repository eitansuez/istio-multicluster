# Istio Multi-cluster on separate networks

Not shown: I created two VPCs, one named network1 in the us-central1 region, and the other network2 in us-east1.

## Create the clusters

```shell
gcloud container clusters create central-cluster \
  --cluster-version latest \
  --machine-type "e2-standard-2" \
  --num-nodes "3" \
  --network "network1" \
  --subnetwork "network1-subnet" \
  --region "us-central1"
```

```shell
gcloud container clusters create east-cluster \
  --cluster-version latest \
  --machine-type "e2-standard-2" \
  --num-nodes "3" \
  --network "network2" \
  --subnetwork "network2-subnet" \
  --region "us-east1"
```

## Create common root of trust

See [these instructions](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/).

### Create the certs and keys

Make the root ca first.

Then:

```shell
make -f ../tools/certs/Makefile.selfsigned.mk central-cluster-cacerts
```

and:

```shell
make -f ../tools/certs/Makefile.selfsigned.mk east-cluster-cacerts
```

## Create the secrets

Create the `istio-system` namespace and create secret in each kube context.

```shell
kubectx gke_eitan-tetrate_us-east1_east-cluster
pushd east-cluster
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=ca-cert.pem \
      --from-file=ca-key.pem \
      --from-file=root-cert.pem \
      --from-file=cert-chain.pem
popd
```

and..

```shell
kubectx gke_eitan-tetrate_us-central1-a_central-cluster
pushd central-cluster
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
      --from-file=ca-cert.pem \
      --from-file=ca-key.pem \
      --from-file=root-cert.pem \
      --from-file=cert-chain.pem
popd
```

## Configure kube context env vars

```shell
export CTX_CLUSTER1=gke_eitan-tetrate_us-central1-a_central-cluster
```

and

```shell
export CTX_CLUSTER2=gke_eitan-tetrate_us-east1_east-cluster
```

## Main course

Follow instructions on [this page](https://istio.io/latest/docs/setup/install/multicluster/primary-remote_multi-network/)

## Test

Deploy helloworld and sleep samples to each cluster in the default namespace (labeled for istio injection).

For complete details, see [these instructions](https://istio.io/latest/docs/setup/install/multicluster/verify/).

### Setup

In each cluster:

- label default ns with istio-injection=enabled
- deploy sleep
- deploy helloworld service only

Then on cluster1 (central), deploy helloworld-v1 only.
And on cluster2 (east), deploy helloworld-v2 only.

### Call the helloworld service from sleep

From cluster1.sleep, calls to helloworld service should round-robin between v1 and v2:

```shell
k exec sleep-69cfb4968f-5spv2 -it -- curl helloworld:5000/hello
```

Likewise from cluster2.sleep.  This indicates that both endpoints are associated with the helloworld service, despite the fact that they live in separate clusters.

### Note IP Addresses:

Cluster 1:

```console
NAME                             READY   STATUS    RESTARTS   AGE     IP
helloworld-v1-77cb56d4b4-8q7jt   2/2     Running   0          8m53s   10.60.2.9
```

```shell
k get svc -n istio-system
```

Output:

```console
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.64.13.164   34.67.135.88   15021:30671/TCP,15443:31169/TCP,15012:32203/TCP,15017:30977/TCP   140m
istio-ingressgateway    LoadBalancer   10.64.14.243   34.71.79.179   15021:30969/TCP,80:32226/TCP,443:30675/TCP                        140m
istiod                  ClusterIP      10.64.6.125    <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP                             140m
```

Cluster 2:

```console
NAME                             READY   STATUS    RESTARTS   AGE   IP
helloworld-v2-79bf565586-xhm4p   2/2     Running   0          11m   10.124.6.4
```

```shell
k get svc -n istio-system
```

Output:

```console
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.128.9.199    34.74.100.144   15021:32281/TCP,15443:30732/TCP,15012:30305/TCP,15017:32378/TCP   114m
istiod                  ClusterIP      10.128.14.225   <none>          15012/TCP,443/TCP                                                 115m
```

### Look at the endpoints

From cluster1 to cluster2:

```shell
istioctl proxy-config endpoint sleep-69cfb4968f-h2njl --cluster "outbound|5000||helloworld.default.svc.cluster.local"
```

Output:

```console
ENDPOINT                STATUS      OUTLIER CHECK     CLUSTER
10.60.2.9:5000          HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
34.74.100.144:15443     HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
```

Note: the 10.60.2.9 address is the pod ip of the local helloworld workload, while the 34.74.100.144 address is that of the east-west gateway on cluster2.

From cluster2 to cluster1:

```shell
istioctl proxy-config endpoint sleep-69cfb4968f-5spv2 --cluster "outbound|5000||helloworld.default.svc.cluster.local"
```

Output:

```console
ENDPOINT               STATUS      OUTLIER CHECK     CLUSTER
10.124.6.4:5000        HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
34.67.135.88:15443     HEALTHY     OK                outbound|5000||helloworld.default.svc.cluster.local
```

Note: the 10.124.6.4 address is the local pod ip for helloworld (cluster2), while the 34.67.135.88 address is that of the east-west gateway on cluster1.

### Observations

1. The main observation is that this exercise confirms that service discovery is mesh-wide, even in the face of workloads residing in separate clusters.
Further, service discovery and propagation of that information to the envoys happens automatically and transparently, as it should, just like in the simpler case of workloads residing in a single-cluster.

1. If I were to scale helloworld-v1 on cluster1 to 2 replicas, the number of endpoints for the helloworld cluster on the sleep pod in cluster2 would not change, because the two endpoints in cluster1 both have technically the same target address, the east-west gateway on cluster1.

    The inference here is that the east-west gateway (on cluster1) receiving the request from cluster2 will load-balance requests across the two helloworld instances residing on its side, on its cluster.
