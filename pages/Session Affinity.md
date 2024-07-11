public:: true
tags:: Kubernetes, Kubernetes Service, iptables

- ## 用途
  讓同一個 Client 來的連接都分配到同一個 Pod
- ## Configuration
  1. None: 不作用
  2. ClientIP: 當 Client IP 相同時導到同一個 EndPoints
  如果使用 [[NodePort]] 或 [[Ingress]] 可能會拿到都是 load-balancer 或 [[Ingress]] controller 的 IP
- ## How