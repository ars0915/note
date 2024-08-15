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
	- deployment name: `my-deployment`
	-