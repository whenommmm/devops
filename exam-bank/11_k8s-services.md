# 11 · Kubernetes Services

*Questions from `session-11-kubernetes-services.md`.*

## Part A · MCQs (cover the bold, answer, then check)

1. Why are Kubernetes Services used?  →  **A. Provide stable access to a set of Pods**
2. Which Service type is typically reachable only within the cluster?  →  **A. ClusterIP**
3. Which Service type exposes a port on each node?  →  **A. NodePort**
4. Which Service type commonly provisions an external load balancer on supported clouds?  →  **A. LoadBalancer**
5. How does a Service identify backend Pods?  →  **A. Label selector**
6. What does a Service port targetPort refer to?  →  **C. Port on backend Pods**
7. What is the role of kube-proxy in many clusters?  →  **D. Implement Service traffic routing rules**
8. Which DNS name form can identify a Service across namespaces?  →  **C. service.namespace.svc.cluster.local**
9. What does a headless Service commonly set?  →  **D. clusterIP: None**
10. Which command lists Services?  →  **C. kubectl get svc**

## Part B · Labs

*(lab answers being written)*
