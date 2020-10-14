# HAProxy ingress controller for Kubernetes
HAProxy ingress controller is a Kubernetes resource that routes traffic from outside your cluster to services within the cluster. this ingress controller uses the HAProxy load balancer.

# Metrics
The HAProxy ingress instruments Prometheus metrics on the following scopes:
- Global
- Frontend
- Backend
- Server (disabled in the configuration provided)

For a complete list of the metrics exported, consult the [HAProxy exporter documentation](https://github.com/haproxy/haproxy/blob/master/contrib/prometheus-exporter/README)

# Number of time series generated
Usually, for the configuration provided, the number of time series is approximately:
- 150 x number of ingress pods
- 50 x number of ingress pods x ingress resources

For further information, consult [HAProxy Kubernetes ingress controller](https://github.com/haproxytech/kubernetes-ingress).

# Attributions
Configuration files and dashboards maintained by [Sysdig team](https://sysdig.com/).

Using [HAProxy Kubernetes ingress controller](https://github.com/haproxytech/kubernetes-ingress) with Apache 2.0 license.