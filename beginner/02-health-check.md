# Health check  

+ `liveness check` : thăm dò để thông báo tới kubelet khi nào thì cần restart pod
+ `readness check` : thăm dò để thông báo tới kubelet biết container đã sẵn sàng để tiếp nhận traffic.  

## Liveness  

```yml
spec:
    containers:
        - name: test
          image: ...
          args: 
          livenessProbe:
            exec:
                command:
                - cat
                - /tmp/healthy
            initialDelaySeconds: 5
            periodSeconds: 5
```

Chúng ta cũng có thể dùng `httpGet` thay thế cho `exec`:  

```yml
httpGet:
    path: /health
    port: 3000
    httpHeaders:
        - name: Custom-Header
          value: Awesome
```

+ `tcpSocket` thiết lập tcp connect tới port chỉ định của container:  

```yml
tcpSocket:
    port: 8080
```

## Readness

Trong trường hợp ứng dụng của chúng ta cần làm một công việc nào đó lúc khởi động, mà tốc độ phụ thuộc vào nhiều yếu tố, chúng ta dùng `readness` để kiểm tra khi nào thì container sẵn sàng để nhận traffic.  

```yml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Các phương thức vẫn giống với `livenessProbe`.  
