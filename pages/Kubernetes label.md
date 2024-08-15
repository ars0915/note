public:: true
tags:: Kubernetes

- Deployment yaml
  
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
  	name: my-deployment
      labels:
      	app: my-app
  spec:
  	replicas: 3
      selector:
      	matchLabels:
          	app: my-app
      template:
      	metadata:
          	labels:
              	app: my-app
  		spec:
          	containers:
              - name: my-app-container
                image: my-app-image
  ```
	- Deployment name: `my-deployment`
	- Deployment labels: `{app: "my-app"}`
	- Deployment 管理 label 為 `{app: "my-app"}` 的 Pod (`spec.selector.matchLabels`)
	-