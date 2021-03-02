# RBAC  

`Role based access control` phân quyền truy cập cho một user, chỉ được truy cập vào những tài nguyên chỉ định.  

Chúng ta sẽ định nghĩa các `role` và sử dụng `rolebinding` để bind một role vào 1 user.  

## Create k8s user  

Trước hết, với tài khoản có quyền truy cập vào cluster, tiến hành trích xuất `ConfigMap` đã có của `aws-auth` trong namespace `kube-system`.  

```bash
kubectl get configmap -n kube-system aws-auth -o yaml > aws-auth.yaml
```

Sau đó, thêm một data mapping trong file `aws-auth.yaml`.  

```bash
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/rbac-user
      username: rbac-user
EoF
```

Apply configMap vừa tạo :  

```bash
kubectl apply -f aws-auth.yaml
```

Với user này, chúng ta chưa thực hiện cấp phát quyền cho nó nên không thể làm được gì.  

## Tạo role  

Chúng ta sẽ tiến hành tạo role để cấp quyền cho user `rbac-user` vừa tạo ở bước trước.  

+ `rbacuser-role.yaml`:  

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

+ `rbacuser-role-binding.yaml`:  

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: User
  name: rbac-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Ta tạo một role `pod-reader` có quyền `get`, `list`, `watch` tới các `pods`, `deployments` trong namespace `rbac-test`. Nếu muốn một role lên toàn bộ cluster thì dùng `ClusterRole`.  
Sau đó, tạo rolebinding `read-pods` để gán role vừa tạo với `rbac-user` <- user mà ta muốn cấp quyền.  
