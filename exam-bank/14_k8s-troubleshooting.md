# 14 · Kubernetes troubleshooting

*Questions from `session-14-kubernetes-troubleshooting.md`.*

## Part A · MCQs (cover the bold, answer, then check)

1. Which command gives a quick overview of Pod status?  →  **A. kubectl get pods**
2. Which command displays detailed Pod events and conditions?  →  **D. kubectl describe pod**
3. Which command reads a container's logs?  →  **D. kubectl logs**
4. Which command runs a command inside a running container?  →  **A. kubectl exec**
5. What often causes ImagePullBackOff?  →  **D. An image cannot be downloaded**
6. What often causes CrashLoopBackOff?  →  **B. A container repeatedly exits or crashes**
7. What is a common reason for a Pod to remain Pending?  →  **D. Insufficient schedulable resources**
8. Where can you see recent scheduling or pull errors?  →  **A. Kubernetes Events**
9. What should you check if a Service has no endpoints?  →  **B. Whether its selector matches ready Pods**
10. What is the final step after fixing an incident?  →  **C. Verify the workload behaves correctly**

## Part B · Labs

*(lab answers being written)*
