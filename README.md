

```markdown
# 🚀 EKS Cluster Auto-Scaler Setup

## 🌍 Real-World Scenario

You're a DevOps engineer managing a Kubernetes workload on AWS EKS. To save costs and improve performance, you want your cluster to automatically scale nodes up or down depending on CPU or memory usage. This guide walks you through enabling **Cluster Autoscaler** on EKS and deploying a real microservices app to trigger scaling.

---

## 📁 Project Structure

```bash
eks-cluster-autoscaler/
├── autoscaler/
│   └── ca.yaml                  # Cluster Autoscaler Kubernetes manifest
├── policies/
│   └── AmazonEKSClusterAutoscalerPolicy.json  # IAM policy for Autoscaler permissions
├── README.md
```

---
## file-setup.py

```python
import os

# Define base path
base_path = "/home/lilia/VIDEOS/eks-cluster-autoscaler"

# Define file paths and contents
files = {
    "autoscaler/ca.yaml": """\
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.26.2
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/clusterautoscaler
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
""",
    "policies/AmazonEKSClusterAutoscalerPolicy.json": """\
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions",
                "eks:DescribeNodegroup"
            ],
            "Resource": "*"
        }
    ]
}
"""
}

# Create directories and files
for relative_path, content in files.items():
    file_path = os.path.join(base_path, relative_path)
    os.makedirs(os.path.dirname(file_path), exist_ok=True)
    with open(file_path, "w") as f:
        f.write(content)

"Files created successfully."


```


---
## ✅ Step-by-Step Setup

### 1️⃣ Create an EKS Cluster

```bash
eksctl create cluster \
  --name clusterautoscaler \
  --node-type t2.large \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --region us-east-1 \
  --zones=us-east-1a,us-east-1b,us-east-1c
```

> This sets up an EKS cluster named `clusterautoscaler` with an autoscaling node group.
> Creates a highly available EKS cluster named clusterautoscaler.
> Auto Scaling Group with 2–3 nodes (t2.large) across 3 AZs.
---

### 2️⃣ Update kubeconfig to Use the New Cluster

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name clusterautoscaler
```

> Configures your local `kubectl` to talk to the new cluster.

---

### 3️⃣ Create IAM Policy for Cluster Autoscaler

Save the following to `policies/AmazonEKSClusterAutoscalerPolicy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the policy in AWS:

```bash
aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://policies/AmazonEKSClusterAutoscalerPolicy.json
```

---

### 4️⃣ Attach IAM Policy to Node Group Role

```bash
NODE_ROLE_NAME=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName | contains("nodegroup")) | .RoleName')

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKSClusterAutoscalerPolicy \
  --role-name $NODE_ROLE_NAME
```

> Ensure you replace `<ACCOUNT_ID>` with your actual AWS account ID.
>  What it does: Gives your worker nodes permission to perform scaling actions.

---

### 5️⃣ Apply Cluster Autoscaler Manifest

Download and apply the autoscaler:

```bash
curl -O https://raw.githubusercontent.com/ooghenekaro/cluster-autoscaler-yaml-sample/main/ca.yml
kubectl apply -f ca.yml
```
Or clone and apply manually:
```bash
git clone https://github.com/ooghenekaro/cluster-autoscaler-yaml-sample.git
kubectl apply -f cluster-autoscaler-yaml-sample/ca.yml

```
Make sure you replace this line with your actual cluster name:

```yaml
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/clusterautoscaler
```
🔍 What it does:
 - Creates service account, roles, role bindings.
 - Deploys the Cluster Autoscaler deployment in kube-system.
 - Sets up autoscaler to detect under/overutilized nodes.

 Verify the Cluster Autoscaler Behavior
```bash
kubectl get deployment cluster-autoscaler -n kube-system
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```
 🔍 What it does:
 - Verifies autoscaler is running and logs scaling decisions.
   
---

### 6️⃣ Deploy Google’s Online Boutique App to Trigger Scaling

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
```

> This will trigger node scaling due to increased resource requests from the microservices.

---

### 7️⃣ Monitor Node Scaling

```bash
kubectl get nodes -w
```

> Watch the nodes scale up or down depending on load.

---

### 🧹 Optional: Delete the Cluster

```bash
eksctl delete cluster \
  --name clusterautoscaler \
  --region us-east-1
```

---

## 🧠 Key Concepts

| Component           | Description                                                       |
|--------------------|-------------------------------------------------------------------|
| `eksctl`           | CLI for easy EKS cluster setup                                    |
| IAM Policy         | Grants permissions to the autoscaler to manage EC2 nodes          |
| `ca.yaml`          | Manifest to deploy autoscaler into the cluster                    |
| Online Boutique    | Real-world app to generate load and test auto-scaling             |

---

## 📎 References

- 📘 [Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws)
- 📘 [Online Boutique Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo)

---

