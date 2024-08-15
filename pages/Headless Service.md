public:: true
tags:: Kubernetes, Kubernetes Service

- # What is Headless Service
	- **將 spec.clusterIP 設定成 None**。
	- 沒有 ClusterIP，因此存取 service 時，k8s DNS 就沒有任何 ClusterIP 的資訊可以回應給 client
	- 若有搭配 Label Selectors，k8s 就會建立相對應的 endpoint，而存取 service 時，k8s DNS 就會直接回應 endpoint list 的資訊(A record)，因此 client 可以**使用 service domain name 直接存取到 pod**，這也意外著headless service並沒有負載平衡。
	- 若是沒有搭配 Label Selector，就沒有 ClusterIP 也沒有對應的 Endpoint。
- # Example
  
  ```yaml
  kind: Service
  apiVersion: v1
  metadata:
    name: nginx-service
  spec:
    selector:
      app: nginx_test
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    clusterIP: None
  ```
-