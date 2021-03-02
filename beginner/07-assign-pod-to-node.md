# Assigning pods to nodes

Thông thường, pods sẽ được tạo ngẫu nhiên trong các node của cluster, tuy nhiên trong nhiều trường hợp chúng ta muốn chỉ định pod chạy trên node nào đó.  

## nodeSelector  

`nodeSelector` là cách đơn giản nhất để thực hiện công việc này.  

Đầu tiên, thêm nhãn cho node :  

```bash
kubectl label nodes ${FIRST_NODE_NAME} disktype=ssd
```

Kiểm tra label vừa gán :  

```bash
kubectl get nodes --selector disktype=ssd
```

Sau đó, chúng ta có thể gán pod vào node đã được đánh label bằng `nodeSelector`:  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

## Inter-pod Affinity and anti-affinity  

+ quyết định triển khai pod dựa trên nhãn của các pods đã chạy trong nodes hơn là nhãn của node, chúng ta sẽ triển khai các ràng buộc, điều kiện để quyết định pod sẽ chạy trong node nào.  

Có 2 loại :  

+ `requiredDuringSchedulingIgnoredDuringExecution` : bắt buộc, Ex: chạy pod A trong node có pod B đang chạy để chia sẻ tài nguyên  
+ `preferredDuringSchedulingIgnoredDuringExecution`: sẽ cố gắng thực hiện nhưng không đảm bảo. Ex: phân tán các pod ra các node khác nhau nếu thỏa mãn, tuy nhiên không bắt buộc nếu thiếu node  

Example:  

+ `pods/pod-with-node-affinity.yaml`:  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

Phân tích :  

+ `podAffinity`, `podAntiAffinity` : chỉ định pod affinity  
+ `requiredDuringSchedulingIgnoredDuringExecution`: bắt buộc thực hiện  
+ [`topologKey`](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/):  là key trong labels của node. Nếu hai node được đánh dấu cùng 1 key, và có cùng giá trị cho nhãn đó, thì hệ thống sẽ coi rằng các node chung 1 topology, từ đó phân bổ đều các pod vào các miền topology khác nhau.  
+ `labelSelector`: định nghĩa cách thức chọn pod.  

Pod trên có ý nghĩa :  

+ `podAffinity`: pod phải được vào các node có pod với nhãn là `security=S1`
+ `podAntiAffinity`: pod nên được ưu tiên vào các node không chứa các pod có nhãn là `security=S2`. Dựa trên tổng `weight`, chọn node có `weight` cao nhất để chạy pod.  

Việc phân bố đều thì sẽ do `topologyKey` đảm nhiệm rồi.  

`Note`: 

+ về tính đối xứng của 2 loại trên, tham khảo tại [ví dụ](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md#a-comment-on-symmetry) của kubernetes.  
  + giả sử chúng ta muốn deploy port `S1` tuân theo `RequiredDuringScheduling` anti affinity, mà trong luật đó không cho `S1` chạy chung với `S2`, mặc dù `S2` không có luật anti affinity, hoặc `S2` tạo sau `S1` nhưng vẫn phải tuân theo luật anti affinity của `S1`.  
    + Nếu trong node không có `S1` hoặc `S2` thì tùy ý
    + Nếu trong node đang có `S1` thì dù `S2` tạo sau cũng không được chạy trong node này.  
  + `RequiredDuringScheduling` affinity lại không có tính chất đối xứng. Giả sử `S1` có `RequiredDuringScheduling` affinity bắt buộc phải chạy với `S2`
    + nếu node rỗng, `S2` có thể khởi tạo, `S1` thì không
    + nếu node có `S2` đang chạy, thì `S1` có thể khởi tạo
    + nếu node đang có `S1`+`S2` chạy mà `S1` ngừng, `S2` vẫn tiếp tục chạy
    + nếu node đang có `S1`+`S2` chạy mà `S2` ngừng thì `S1` cũng bị ngừng  

## Anothor practice example  

Giả sử chúng ta chuẩn bị chạy một hệ thống sử dụng `redis` để cache. Ta sẽ phải thiết kế sao cho các `web server` phải được gần các `redis-server` nhất, tối ưu đường truyền. Tốt nhất nó phải được ở cùng node.  

Chúng ta sẽ tiến hành cấu hình `affinity` để điều này xảy ra.  

+ `redis-with-node-affinity.yaml` :  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

Redis server được thiết lập để không có `redis-server` nào được ở cùng trên 1 node.  

+ `web-with-node-affinity.yaml`:  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

+ `podAntiAffinity`:  web-server không được chạy trên cùng 1 pod
+ `podAffinity`: web-server phải chạy trên node có ít nhất 1 `redis-server`  

Sau khi triển khai, hệ thống sẽ có dạng :  

```bash
node-1 - webserver-1 - cache-1
node-2 - webserver-2 - cache-2
node-3 - webserver-3 - cache-3
```

Do tính chất đối xứng của `podAntiAffinity` cho nên chỉ cần tạo anti affinity tại `web-server`.  
Do tính bất đối xứng của `podAffinity` nên khi mà một `redis-server` bị ngưng thì `web-server` cũng bị ngưng và cùng khởi tạo lại, cho nên luôn tồn tại `redis-server` trên cùng 1 node với `web-server`.  

## Ref  

[ref](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)  
