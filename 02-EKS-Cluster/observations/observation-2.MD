**1. Managed Node Group (system workloads) with taint**  
**2. Karpenter NodePool (workloads only)**  
**3. AWS Load Balancer Controller Helm release with tolerations**  
**4. Proper separation so system pods never land on Karpenter nodes**
I’ll make it modular and fully declarative.  

1️⃣ **Managed Node Group (MNG) — system nodes only)**  
```text
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
