## 1️⃣ Terraform Helm provider basic example  
```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}
```
Then your **AWS Load Balancer Controller release:**  
```hcl
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.5.3"   # Use the version you need

  create_namespace = false

  # Required cluster-specific values
  set {
    name  = "clusterName"
    value = var.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  # Optional values via file
  values = [file("${path.module}/alb-values.yaml")]
}
```

2️⃣ **alb-values.yaml example**  
```yaml
replicaCount: 2

tolerations:
  - key: "node-role"
    operator: "Equal"
    value: "system"
    effect: "NoSchedule"

nodeSelector:
  node-role: system
```
✅ This will ensure the controller **schedules only on your MNG nodes.**  

3️⃣ **Key notes for Terraform Helm provider**  
**1.** `values` vs `values_files`  
- `values` takes a **list of strings**, each one being a YAML file via `file()`
- `values_files` was used in older versions, now `values` is recommended.  
**2. Override precedence**  
- `set` > `values` 
- So clusterName, SA, etc. are set via `set`, rest via `values`.  
**3. Namespace**  
- Make sure `namespace = "kube-system"` exists before applying.  
- You can also do `create_namespace = true`, but usually kube-system exists.  

4️⃣ **Verify after Terraform apply**  
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -o wide
```
- NODE → should be a Managed Node Group node (tainted `node-role=system`)
- STATUS → Running  

5️⃣ **Optional: dynamic Helm values** 
If you want to **programmatically set tolerations**, you can do:  
```json
values = [
  yamlencode({
    replicaCount = 2
    tolerations = [
      {
        key    = "node-role"
        operator = "Equal"
        value  = "system"
        effect = "NoSchedule"
      }
    ]
    nodeSelector = {
      "node-role" = "system"
    }
  })
]
```
This avoids maintaining a separate YAML file if you prefer fully Terraform-managed config.  

💡 Best practice:
- Keep `alb-values.yaml` in version control (Helm values for system controllers)
- Pass via `values = [file(...)]` to Helm provider
- Combine with `set` for cluster-specific overrides
This keeps your **Terraform plan fully declarative and repeatable.**  
