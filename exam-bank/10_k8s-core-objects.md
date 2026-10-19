# 10 · Pods, ReplicaSets & Deployments

*Questions from `session10-k8s-core-objects.md`.*

## Part A · MCQs (cover the bold, answer, then check)

1. What is the purpose of a Deployment?  →  **B. Manage desired replicas and rolling updates**
2. What does a ReplicaSet ensure?  →  **A. A desired number of matching Pods are running**
3. Which field commonly sets Deployment replica count?  →  **D. spec.replicas**
4. What does a Pod label help with?  →  **C. Selecting and organizing resources**
5. What is a label selector used for?  →  **B. Matching resources with specified labels**
6. Which command scales a Deployment?  →  **C. kubectl scale deployment**
7. What happens when a Deployment Pod crashes?  →  **A. A controller works to restore desired state**
8. Which command views rollout status?  →  **B. kubectl rollout status deployment/name**
9. What does a rolling update do?  →  **A. Gradually replace old Pods with new ones**
10. Which resource usually creates and manages ReplicaSets?  →  **D. Deployment**

## Part B · Labs

*(lab answers being written)*
