
# Digital Ocean Kubernetes Througput Test
With this repo I provide a way to reproduce my thoughput load testing setup on a DO k8s cluster.


## Server Configuration

Create a cluster with
node pool with 6 nodes (s-2vcpu-4gb)

**Setup Loadbalancer and ingress**

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.1/deploy/static/provider/do/deploy.yaml
```

Add the following annotations to the load balancer config

```
service.beta.kubernetes.io/do-loadbalancer-enable-backend-keepalive: "true"
service.beta.kubernetes.io/do-loadbalancer-size-unit: "2"
service.beta.kubernetes.io/do-loadbalancer-protocol: "http"
service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
service.beta.kubernetes.io/do-loadbalancer-tls-passthrough: "true"
service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"
service.beta.kubernetes.io/do-loadbalancer-http-ports: "80"
service.beta.kubernetes.io/do-loadbalancer-algorithm: "least_connections"
```

You can do this by running:

```
kubectl edit service/ingress-nginx-controller -n ingress-nginx
```

scale ingress controller to two pods:

```
kubectl scale --replicas=2 deployment/ingress-nginx-controller -n ingress-nginx
```
make sure they are not scheduled on the same host!

**Deploy the nginx backend server**

```
kubectl apply -f server-cluster
```
Check that all 4 pods are running. The're configured with an podAntiAffinity rule that will prevent them from being scheduled on the same nodes as the ingress-controller pods.

```
> kubectl get pods                                                                                                                                                                      
NAME                               READY   STATUS    RESTARTS   AGE
nginx-load-test-6c5579597c-2w98h   1/1     Running   0          12s
nginx-load-test-6c5579597c-5zl2v   1/1     Running   0          12s
nginx-load-test-6c5579597c-kb5c7   1/1     Running   0          12s
nginx-load-test-6c5579597c-mxm9q   1/1     Running   0          12s
```

**Install Prometheus / Grafana**

```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```


## Client Configuration

Create a cluster with
node pool with 3 nodes (General purpose - 8GB total)

**Deploy bombardier client pods**

```
kubectl apply -f client-cluster
```



## Testing

**On the server cluster**

1. Portforward grafana pod on port 3000. eg `kubectl port-forward kube-prometheus-stack-grafana-564d999d8-fj8mg 3000:3000`
2. Login with admin / prom-operator
3. Open dashboard: Kubernetes / Compute Resources / Workload
4. Filter by type deployment > nginx-load-test
5. Monitor Transmit bandwidth

**On the client cluster**


1. Load test with multiple pods, whilst monitoring grafana, first starting with one and build up to three
2. Exec in container: `kubectl exec -ti bombardier-65c5d96f95-6dw45 -- ash`
3. Start load test: `bombardier -c 7 -d 30m http://<loadbalancerIP>`


## Results / Discussion
When running this test all 4 nginx pods, I'm getting a throughput around 120 MBp/s (960Mbp/s).

As described above the nginx-ingress pods are running on seperate nodes from the nginx pods. With load balancing algorythm set to round robin means that each nginx pod will receive 1/4 of the load.

As there is also almost no ingress the egress should be able to reach a throughput of 4Gbp/s as each node should be able provide 2Gb/s each. This is not happening.

I hope this repo helps you to replicate my results.

