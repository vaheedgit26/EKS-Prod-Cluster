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
  
2️⃣ **kube-proxy**
✅ **Handled automatically**
- Runs as a DaemonSet  
- EKS manages tolerations internally  
No action needed.  

3️⃣ **AWS VPC CNI (aws-node)**  
✅ **Handled automatically**  
- DaemonSet  
- Already tolerates most taints  
Just verify it schedules  

🔹 **AWS / Infra controllers (YOU must configure)**  
4️⃣ **AWS Load Balancer Controller**  
- Helm-installed  
- **Needs Toleration**   
```yaml
tolerations:
  - key: node-role
    value: system
    effect: NoSchedule

nodeSelector:
  node-role: system
```

5️⃣ EBS CSI Controller  
