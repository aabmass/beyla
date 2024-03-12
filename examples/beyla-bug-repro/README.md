# Reproducer for Beyla bug

## Create new namespace

```sh
kubectl create namespace beyla-repro
```

## Start [BookInfo App](https://istio.io/latest/docs/examples/bookinfo/)

```sh
kubectl apply -n beyla-repro -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
```

## Apply local resources

```sh
kubectl apply -n beyla-repro -f .
```

Creates
1. A python script that sends requests to "Product page" at ~2 QPS
1. Beyla running as a Daemonset and rbac config
1. OTel collector configured to log all **server** spans

## Watch OTel collector logs for the bad span

This is the expected service topology:

![](https://istio.io/latest/docs/examples/bookinfo/noistio.svg)

Notice `Reviews-v2` calls `Ratings`.

```sh
kubectl logs -n beyla-repro deployments/otel-collector-deployment | grep server.address | sort | uniq -c   
      2      -> server.address: Str(10.4.0.122)
      8      -> server.address: Str(10.4.1.185)
      4      -> server.address: Str(10.4.1.186)
      4      -> server.address: Str(10.4.1.187)                                                                                                  
      2      -> server.address: Str(10.4.1.188)
      8      -> server.address: Str(10.4.1.189)
      4      -> server.address: Str(10.8.11.125)
```

`10.8.11.125` is the unexpected ClusterIP as a `server.address`.

I'm seeing the bug in the first few minutes of Beyla running so you may need to restart the
Daemonset to get a repro (or if the collector misses the first few spans).
```sh
kubectl -n beyla-repro rollout restart deployment otel-collector-deployment
kubectl -n beyla-repro rollout restart daemonset beyla-agent 
```
