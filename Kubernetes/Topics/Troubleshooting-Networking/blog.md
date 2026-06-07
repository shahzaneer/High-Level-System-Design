# Troubleshooting: Networking & Service Issues

Networking issues are among the hardest to debug in Kubernetes because they span multiple layers: pod networking, service abstraction, kube-proxy/iptables, DNS, ingress, and network policies.

## Service Connection Issues

```bash
# Problem: Can't reach service from another pod
# Method: Debug from a temporary pod in the same namespace

# 1. Check the service exists and has endpoints
kubectl get svc order-service
kubectl get endpoints order-service
# If endpoints is <none> → service selector doesn't match any pods

# 2. Check service selector vs pod labels
kubectl get svc order-service -o jsonpath='{.spec.selector}'
kubectl get pods -l app=order-service
# Labels must match!

# 3. Test DNS resolution
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup order-service
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup order-service.production.svc.cluster.local

# 4. Test direct connection
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -v http://order-service:8080/health

# 5. Test pod IP directly (bypass service)
POD_IP=$(kubectl get pod order-service-abc123 -o jsonpath='{.status.podIP}')
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -v http://${POD_IP}:8080/health
# If pod IP works but service doesn't → kube-proxy/iptables issue
```

## DNS Issues

```bash
# Common CoreDNS problems
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# DNS resolution test
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default
# Should return the cluster IP for kubernetes service

# CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Common: /etc/resolv.conf in pod has wrong nameserver
kubectl exec <pod> -- cat /etc/resolv.conf
# Should show: nameserver <cluster-dns-ip>
```

## Network Policy Debugging

```bash
# Problem: Pod A can't reach Pod B, but service works from other pods
# Check if NetworkPolicy is blocking

kubectl get networkpolicies -n production
kubectl describe networkpolicy <name> -n production

# Quick test: temporarily label pod as allowed
kubectl label pod <source-pod> test-access=true -n production

# Common mistake: egress to DNS blocked
# ALWAYS allow egress to kube-dns on UDP 53 and TCP 53!
```

## Ingress Issues

```bash
kubectl get ingress -A
kubectl describe ingress order-api -n production

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check TLS certificate
kubectl get certificate -n production   # If using cert-manager
kubectl describe certificate api-tls -n production

# Test ingress directly from outside
curl -v -H "Host: api.example.com" http://<ingress-controller-ip>/api/orders
```

## kube-proxy Debugging

```bash
# kube-proxy writes iptables rules for services
# On node, check iptables rules
iptables -t nat -L KUBE-SERVICES -n | grep order-service

# Check kube-proxy mode
kubectl logs -n kube-system <kube-proxy-pod> | grep "Using"
# "Using iptables Proxier" or "Using ipvs Proxier"
```

## Quick Diagnostic Checklist

```
1. Can't reach service at all:
   [ ] Service exists? kubectl get svc
   [ ] Service has endpoints? kubectl get endpoints
   [ ] Pod labels match service selector?
   [ ] Pod is ready? (readinessProbe passing)
   [ ] NetworkPolicy blocking? kubectl get networkpolicies
   [ ] Ports match? (service port vs container port)

2. Service works, but slowly:
   [ ] DNS resolution slow? nslookup timing
   [ ] Too many iptables rules? (scale issue)
   [ ] kube-proxy in wrong mode?

3. External access (Ingress/LB) broken:
   [ ] Ingress resource exists and configured?
   [ ] Ingress controller running?
   [ ] TLS certificate valid?
   [ ] Load balancer health checks passing?
```
