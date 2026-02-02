1️⃣ **Install Kyverno using kubectl (quickest)**  
This is the easiest way to get started:  
```bash
# 1. Create the kyverno namespace
kubectl create namespace kyverno
----------------------------------------------------------------------------------------------
# 2. Apply the latest stable release
kubectl apply -f https://github.com/kyverno/kyverno/releases/latest/download/release.yaml
```
✅ This will install:   
- Kyverno Deployment in kyverno namespace  
- CRDs for Policies, ClusterPolicies, etc.  

2️⃣ **Install Kyverno using Helm (more production-friendly)**  
**1. Add the Kyverno Helm repo**  
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```
**2. Install kyverno**  
```bash
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set replicaCount=2 \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=512Mi
```
- `replicaCount=2` ensures **high availability**  
- Resource requests/limits are optional but recommended for production  

3️⃣ **Verify Kyverno installation**  
```bash
kubectl get pods -n kyverno
```
You should see something like:  
```bash
NAME                       READY   STATUS    RESTARTS   AGE
kyverno-7c8d5f4b9f-abcde  1/1     Running   0          1m
kyverno-7c8d5f4b9f-fghij  1/1     Running   0          1m
```

4️⃣ **Test Kyverno is working**   
**1. Create a simple test ClusterPolicy:**   
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: test-deny-all
spec:
  validationFailureAction: enforce
  rules:
    - name: deny-app-namespace
      match:
        resources:
          kinds: ["Pod"]
          namespaces: ["default"]
      validate:
        message: "Pods in default namespace are denied"
        deny: {}
```
**2. Apply a pod in default namespace:**  
```bash
kubectl run test-nginx --image=nginx
```
You should see rejected by Kyverno. ✅  

5️⃣ **Next step for your setup**  
After Kyverno is running, you can add the **policy we discussed:**  
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
- Once applied, **any pod in non-kube-system namespace that tries to tolerate `node-role=system` will be denied.**  
- Combined with MNG taint, it guarantees **system pods only run on MNG nodes,** and **apps only run on Karpenter nodes.**    
