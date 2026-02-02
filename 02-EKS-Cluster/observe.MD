## 🔹 kube-system (core) 
1️⃣ CoreDNS  
- **Why**: Cluster DNS, HPA-enabled  
- **How**: EKS Add-on or Helm   

Toleration:  
```yaml
tolerations:
  - key: node-role
    value: system
    effect: NoSchedule
```
Node selector:  
```yaml
nodeSelector:
  node-role: system
```
