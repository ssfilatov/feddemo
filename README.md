# MultiCluster

## Kubefed

### Install Kubefed Control Plane on host cluster

```console
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm install kubefed-charts/kubefed --name kubefed --version=0.1.0-rc6 --namespace kube-federation-system --devel
```

### join host & member clusters into federation

```console
export HOST_NAME=<host name>
export MEMBER_NAME=<member name>
export HOST_CTX=<host ctx>
export MEMBER_CTX=<member ctx>
```

```console
kubefedctl join ${HOST_NAME} --host-cluster-name ${HOST_NAME} --cluster-context ${HOST_CTX} \
    --host-cluster-context ${HOST_CTX} --v=2
kubefedctl join ${MEMBER_NAME} --host-cluster-name ${HOST_NAME} --cluster-context ${MEMBER_CTX} \
    --host-cluster-context ${HOST_CTX} --v=2
kubectl --context=${HOST_CTX} -n kube-federation-system get kubefedclusters
```

### Test on helloworld application

```console
kubectl --context=${HOST_CTX} apply -f samples/demo-namespace.yaml
kubectl --context=${HOST_CTX} apply -f samples/helloworld.yaml

curl $(kubectl --context=${HOST_CTX} get svc -n demo -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'):5000/hello
curl $(kubectl --context=${MEMBER_CTX} get svc -n demo -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'):5000/hello
```

```console
kubectl --context=${HOST_CTX} apply -f samples/replica-scheduling-preference.yaml
```


### Set up external DNS
```console
export HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name --output json --dns-name "demo.com" | jq -r '.HostedZones[0].Id')
aws route53 list-resource-record-sets --output json --hosted-zone-id $HOSTED_ZONE_ID

kubectl --context=${HOST_CTX} --apply -f samples/external-dns.yaml
kubectl --context=${HOST_CTX} --apply -f samples/external-dns-test.yaml

aws route53 list-resource-record-sets --output json --hosted-zone-id $HOSTED_ZONE_ID \
--query "ResourceRecordSets[?Name == 'nginx.external-dns-test.demo.com.']|[?Type == 'A']"
```

for some reason my curl build does not support hostnames in --dns-servers
```console
curl --dns-servers ${DNS_SERVER_IP} nginx.external-dns-test.demo.com.
```

### Set up Multicluster dns

```console
kubectl --context=${HOST_CTX} apply -f samples/multicluster-dns.yaml

dig +short @${DNS_SERVER_IP} helloworld.demo.demo-domain.svc.demo.com
curl --dns-servers ${DNS_SERVER_IP} helloworld.demo.demo-domain.svc.demo.com:5000/hello
```

## Istio

### Shared Control Plane

```console
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem
```

```console
export LOCAL_CLUSTER_CTX=<local ctx>
export REMOTE_CLUSTER_CTX=<remote ctx>

export LOCAL_CLUSTER_NETWORK=local
export REMOTE_CLUSTER_NETWORK=remote

export LOCAL_CLUSTER_NAME=<...>
export REMOTE_CLUSTER_NAME=<...>
```

```console
istioctl --context=${LOCAL_CLUSTER_CTX} manifest apply -f samples/istio-local-cluster.yaml
kubectl --context=${LOCAL_CLUSTER_CTX} -n istio-system get pod
```


```console
export ISTIOD_REMOTE_EP=$(kubectl --context=${MAIN_CLUSTER_CTX} -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "ISTIOD_REMOTE_EP is ${ISTIOD_REMOTE_EP}"
```

```console
kubectl --context=${REMOTE_CLUSTER_CTX} create namespace istio-system
kubectl c--context=${REMOTE_CLUSTER_CTX} create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem
```

```console
istioctl --context=${REMOTE_CLUSTER_CTX} manifest apply -f istio-remote-cluster.yaml
```

```
kubectl --context=${LOCAL_CLUSTER_CTX} apply -f samples/cluster-aware-gateway.yaml
kubectl --context=${REMOTE_CLUSTER_CTX} apply -f samples/cluster-aware-gateway.yaml
istioctl x create-remote-secret --context=${REMOTE_CLUSTER_CTX} --name ${REMOTE_CLUSTER_NAME} | \
    kubectl apply -f - --context=${LOCAL_CLUSTER_CTX}
```

```
kubectl create --context=${REMOTE_CLUSTER_CTX} namespace sample
kubectl label --context=${REMOTE_CLUSTER_CTX} namespace sample istio-injection=enabled
kubectl create --context=${REMOTE_CLUSTER_CTX} -f samples/helloworld/helloworld.yaml -l app=helloworld -n sample
kubectl create --context=${REMOTE_CLUSTER_CTX} -f samples/helloworld/helloworld.yaml -l version=v2 -n sample
```

```console
kubectl create --context=${LOCAL_CLUSTER_CTX} namespace sample
kubectl label --context=${LOCAL_CLUSTER_CTX} namespace sample istio-injection=enabled
```
```console
kubectl create --context=${LOCAL_CLUSTER_CTX} -f samples/helloworld/helloworld.yaml -l app=helloworld -n sample
kubectl create --context=${LOCAL_CLUSTER_CTX} -f samples/helloworld/helloworld.yaml -l version=v1 -n sample
```
```console
kubectl apply --context=${LOCAL_CLUSTER_CTX} -f samples/sleep/sleep.yaml -n sample
kubectl apply --context=${REMOTE_CLUSTER_CTX} -f samples/sleep/sleep.yaml -n sample
```

```console
kubectl apply --context=${LOCAL_CLUSTER_CTX} -f samples/destinationrule.yaml
```

```console
k exec sleep-* -n sample -c sleep -- curl helloworld:5000/hello
istioctl --context=${LOCAL_CLUSTER_CTX} -n sample proxy-config endpoints sleep-* --cluster "outbound|5000||helloworld.sample.svc.cluster.local"
```
