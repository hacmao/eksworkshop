# Kubectl dashboard  

Triển khai dashboard bằng kubectl :  

```bash
export DASHBOARD_VERSION="v2.0.0"

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml
```

Mở proxy cho mạng ngoài có thể truy cập được :  

```bash
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
```

+ Link truy cập :  

```bash
/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

+ Token :  

```bash
aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
```

+ Cleanup :  

```bash
pkill -f 'kubectl proxy --port=8080'

# delete dashboard
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml
```
