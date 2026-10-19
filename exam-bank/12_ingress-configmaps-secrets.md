# 12 · Ingress, ConfigMaps & Secrets

*Questions from `session-12-ingress-configmaps-secrets.md`.*

## Part A · MCQs (cover the bold, answer, then check)

1. What does Ingress commonly define?  →  **A. HTTP/HTTPS routing rules to Services**
2. What is needed to implement Ingress rules?  →  **D. An Ingress controller**
3. What is a ConfigMap intended to store?  →  **D. Non-sensitive configuration data**
4. What is a Secret intended to store?  →  **A. Sensitive configuration values**
5. How can a ConfigMap be consumed by a Pod?  →  **D. Environment variables or mounted files**
6. What does a path rule in Ingress match?  →  **B. The HTTP request path**
7. What does TLS termination usually provide at Ingress?  →  **D. HTTPS encryption for client connections**
8. Which command lists ConfigMaps?  →  **A. kubectl get configmaps**
9. Are Kubernetes Secret values encrypted merely because YAML uses base64?  →  **B. No, base64 is encoding rather than encryption**
10. Which Kubernetes resource commonly holds a TLS certificate and key for Ingress?  →  **C. Secret**

## Part B · Labs

*(lab answers being written)*
