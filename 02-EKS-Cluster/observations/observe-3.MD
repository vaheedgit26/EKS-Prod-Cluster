**1. Managed Node Group (system workloads) with taint**  
**2. Karpenter NodePool (workloads only)**  
**3. AWS Load Balancer Controller Helm release with tolerations**  
**4. Proper separation so system pods never land on Karpenter nodes**  
I’ll make it modular and fully declarative.  

1️⃣ **Managed Node Group (MNG) — system nodes only)**  
```hcl
resource "aws_eks_node_group" "system_nodes" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.name}-system-ng"
  node_role_arn   = aws_iam_role.eks_nodegroup_role.arn
  subnet_ids      = data.terraform_remote_state.vpc.outputs.private_subnet_ids

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 4
  }

  taints {
    key    = "node-role"
    value  = "system"
    effect = "NO_SCHEDULE"
  }

  labels = {
    "node-role" = "system"
  }

  # Optional: AMI type
  ami_type = "AL2023_x86_64_STANDARD"

  depends_on = [aws_iam_role_policy_attachment.eks_worker_node_policy]
}
```

✅ Key points:  
- Taint ensures **only pods** with `node-role=system` toleration can schedule.
- Labels allow `nodeSelector` in Helm charts.

2️⃣ **Karpenter NodePool — workloads only**  
```hcl
resource "karpenter_node_pool" "workload_nodes" {
  name = "workload-nodes"

  template {
    spec {
      node_class_ref {
        group = "karpenter.k8s.aws"
        kind  = "EC2NodeClass"
        name  = "default-ec2nodeclass"
      }

      # No taints → workload pods only
      taints = []
    }

    requirements {
      key      = "kubernetes.io/arch"
      operator = "In"
      values   = ["amd64"]
    }
    requirements {
      key      = "kubernetes.io/os"
      operator = "In"
      values   = ["linux"]
    }
    requirements {
      key      = "karpenter.sh/capacity-type"
      operator = "In"
      values   = ["on-demand"]
    }
  }

  limits {
    cpu = "50"
  }

  disruption {
    consolidation_policy = "WhenEmptyOrUnderutilized"
    consolidate_after    = "30s"
  }
}
```
✅ Key points:  
- Karpenter nodes **cannot schedule system pods** because they have **no toleration for** `node-role=system`
- All workload pods go here by default   
3️⃣ AWS Load Balancer Controller — Helm release with toleration   

`alb-values.yaml:`   
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
Terraform Helm resource:   
```hcl
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.5.3"
  create_namespace = false

  set {
    name  = "clusterName"
    value = aws_eks_cluster.main.name
  }

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  values = [file("${path.module}/alb-values.yaml")]

  depends_on = [aws_eks_node_group.system_nodes]
}
```
✅ Key points:   
- Uses **toleration + nodeSelector** → guaranteed to schedule only on system MNG nodes   
- `depends_on` ensures the node group exists before Helm applies   

4️⃣ **Verification steps after** `terraform apply`   
```bash
# 1. Managed Node Group taints
kubectl get nodes -o wide
kubectl describe node <mng-node-name> | grep Taints

# 2. Load Balancer Controller pods
kubectl get pods -n kube-system -o wide
# Should all be on MNG nodes

# 3. Workload pods (default namespace)
kubectl run test-nginx --image=nginx
kubectl get pods -o wide
# Should only land on Karpenter nodes
```

5️⃣ **Optional: Kyverno enforcement (future-proof)**   
If you want **no human mistakes**, add Kyverno:  
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-system-node-tolerations
spec:
  validationFailureAction: Enforce
  rules:
    - name: only-kube-system-can-tolerate-system-nodes
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Only pods in kube-system may tolerate node-role=system"
        deny:
          conditions:
            all:
              - key: "{{ request.object.metadata.namespace }}"
                operator: NotEquals
                value: kube-system
              - key: "{{ request.object.spec.tolerations[].key || '' }}"
                operator: AnyIn
                value: ["node-role"]
```
✅ Guarantees system nodes cannot be polluted with app pods, even if someone forgets Helm values.   
This gives you full separation:   
| Node type | Taint                         | Allowed pods                            |
| --------- | ----------------------------- | --------------------------------------- |
| MNG       | `node-role=system:NoSchedule` | kube-system only (controllers)          |
| Karpenter | none                          | All workloads (default namespace, apps) |  

| Component      | Role                                                |
| -------------- | --------------------------------------------------- |
| Node taint     | Block unwanted pods from scheduling                 |
| Pod toleration | Allow pod to override the taint                     |
| Kyverno policy | Enforce “only kube-system may have that toleration” |   

So in short:
**Taint + toleration = scheduling control**  
**Kyverno = enforcement guardrail**  
You **still need to add tolerations** to system pods; Kyverno just prevents mistakes in the future.


