public:: true
tags:: Kubernetes, Kubernetes Service, iptables

- `NodePort` 本身包含了 `ClusterIP` 的能力，此外多提供了一種能力讓`非叢集`的應用程式/節點也有辦法存取叢集內的應用程式。 舉例來說，我們可以部屬多個網頁伺服器，然後透過 `NodePort` 的方式讓外部的電腦(瀏覽器）來存取這些在 `kubernetes` 叢集內的網頁伺服器。
-