## Addons
🔹 **kube-system (core)**  
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
Install / upgrade:  
```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f alb-values.yaml \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

5️⃣ EBS CSI Controller  
- Helm or EKS add-on  
Controller pods need toleration:  
```yaml
tolerations:
  - key: node-role
    value: system
    effect: NoSchedule
```   
(Node DaemonSet part is fine.)  

6️⃣ **Metrics Server**  
- Used by HPA
- Often forgotten ❗  
```yaml
tolerations:
  - key: node-role
    value: system
    effect: NoSchedule
```

7️⃣ **ExternalDNS (if used)**
```yaml
tolerations:
  - key: node-role
    value: system
    effect: NoSchedule

nodeSelector:
  node-role: system
```

🔹 **Observability / Ops (depends on your stack)** 
Check if you run any of these in kube-system:  
- Fluent Bit / Fluentd  
- Prometheus Operator  
- CloudWatch Agent  
- Datadog agent (controller part)  
Controllers → **YES, toleration needed**   
DaemonSets → usually OK

🧪 **Validation steps (do this in order)** 
1️⃣ **Verify node taint is present**  
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```
You should see:  
```bash
node-role=system:NoSchedule
```  
on **managed node group nodes only.**  

2️⃣ **Check kube-system pods are running**  
```bash
kubectl get pods -n kube-system -o wide
```
Confirm:  
- STATUS = Running  
- NODE = managed node group nodes  
- No Pending pods  

3️⃣ **Look for scheduling failures (very important)**  
```bash
kubectl get events -n kube-system --sort-by=.lastTimestamp
```
If something is missing toleration, you’ll see:  
```bash
node(s) had taint {node-role: system}, that the pod didn't tolerate
```  

4️⃣ **Verify app pods cannot land on system nodes** 
Create a test pod (default namespace):  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: should-not-run
spec:
  containers:
    - name: nginx
      image: nginx
``` 
Apply:  
```bash
kubectl apply -f test.yaml
```
Expected:  
- Pod schedules on Karpenter node
- Never on MNG

5️⃣ **Verify Karpenter nodes stay workload-only**  
```bash
kubectl get pods -A -o wide | grep <karpenter-node-name>
```
You should see:  
- App pods only  
- No kube-system controllers  

🚨 Common mistakes to watch for  
❌ Forgetting Metrics Server  
❌ Helm chart upgrades without values file  
❌ Installing new add-ons without tolerations  
❌ Assuming kube-system namespace = safe  

🧠 **Golden rule (remember this)**  
**Taints define ownership.**  
**Tolerations grant permission.**  
**Namespaces do nothing.**  
