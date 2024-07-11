## External Name
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
-