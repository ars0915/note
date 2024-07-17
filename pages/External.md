public:: true

- ## External Name
	- 把 Service 導向指定的 DNS name，而不是 service 中 label selector 所設定的 pod
	  ```
	  kind: Service
	  apiVersion: v1
	  metadata:
	    name: my-service
	    namespace: prod
	  spec:
	    type: ExternalName
	    externalName: my.database.example.com
	  ```
	  當 cluster 內部查找 my-service.prod.svc 的時候，k8s DNS service 就只會回應 my.database.example.com 這個 CNAME recrd。
	  但回應 CNAME record 跟其他 type 有何差別? 其實就是當存取 ExternalName type 的 service 時，網路流量的導向是發生在 DNS level 而不是透過 proxying or forwarding 達成的。
	  若要使用 ExternalName，k8s DNS service 的部份僅能安裝 kube-dns，且版本需要是 1.7 以上
- ## External IP
	- 讓使用者指定 service 要黏在哪個 IP 上
	  ```
	  kind: Service
	  apiVersion: v1
	  metadata:
	    name: my-service
	  spec:
	    selector:
	      app: MyApp
	    ports:
	    - name: http
	      protocol: TCP
	      port: 80
	      targetPort: 9376
	    # 此 service 只會將進入 80.11.12.10:80 的 網路流量
	    # 導向後端的 endpoints 
	    externalIPs:
	    - 80.11.12.10
	  ```