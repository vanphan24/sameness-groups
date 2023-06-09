# sameness-groups

# Pre-reqs

You have two Kubernetes clusters available. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.

Note:

If using AKS, you can use the Kubenet CNI or the Azure CNI. The Consul control plane and data plane will use Load Balancers (via Consul mesh gateways)to communicate between Consul datacenters.
Since Load Balancers are used on both control plane and data plane, each Consul datacenter can reside on different networks (VNETS, VPCs) or even different clouds (AWS, Azure GCP, private, etc). No direct network connections (ie VPC/VNET peering connections) are required.

2. Add or update your hashicorp helm repo:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
or
```
helm repo update hashicorp
```
Note:
 
For pre-release beta builds of Consul on Kubernetes images, clone the consul-k8s repo.

```
git clone https://github.com/hashicorp/consul-k8s.git
```
3. You will need a Consul Enterprise license. 

You can request a 30 day trial license here: https://www.hashicorp.com/products/consul/trial

# Deploy Consul on first Kubernetes cluster (dc1).

1. Clone this repo
```
git clone https://github.com/vanphan24/sameness-groups.git
```

2. Nagivate to the **sameness-groups** folder. 

```
cd sameness-groups
```

3. Set environemetal variables for kubernetes cluster dc1 and dc2

```
export dc1=<your-kubernetes context-for-dc1>
export dc2=<your-kubernetes context-for-dc2>
```

4. Set you Consul Enterprise license as an environment variable.

```
export CONSUL_LICENSE=<ADD_YOUR_LICENSE_HERE>
```

5. Add Consul Ent license as a K8s secret

```
kubectl create secret generic license --from-literal=key=$CONSUL_LICENSE --context $dc1
```


6. Set context and deploy Consul on dc1

```
kubectl config use-context $dc1
``` 

```
helm install $dc1 hashicorp/consul --values consul-values-sameness.yaml --set global.datacenter=dc1     
```

or if pointing to pre-reelase K8s chart that is locally clone Consul-k8s repo:
```
helm install $dc1 ../consul-k8s/charts/consul --values consul-values-sameness.yaml --set global.datacenter=dc1                         
```

7. Confirm Consul deployed sucessfully

```
kubectl get pods --context $dc1
NAME                                               READY      STATUS       RESTARTS   AGE
consul-consul-connect-injector-5978975f65-wb6cz       1/1     Running   0             25h
consul-consul-mesh-gateway-f647c58fb-hp65d            1/1     Running   0             25h
consul-consul-server-0                                1/1     Running   1 (24h ago)   25h
consul-consul-webhook-cert-manager-84ffb678cc-rmvkc   1/1     Running   0             25h
```  
Note: Run ```kubectl get crd``` and make sure that exportedservices.consul.hashicorp.com exist.    
If not, you need to upgrade your helm deployment:  
    
```
helm upgrade $dc1 hashicorp/consul --values consul-values-sameness.yaml  
```

8. Deploy both dashboard and counting service on dc1
```
kubectl apply -f countingapp/dashboard.yaml --context $dc1
kubectl apply -f countingapp/counting.yaml --context $dc1
```

9. Using your browser, check the dashboard UI and confirm the number displayed is incrementing. 
   You can get the dashboard UI's EXTERNAL IP address with command below. Make sure to append port :9002 to the browser URL.  
```   
kubectl get service dashboard --context $dc1
```

Example: 
```
kubectl get service dashboard --context $dc1
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
dashboard   LoadBalancer   10.0.179.160   40.88.218.67  9002:32696/TCP   22s
```


![alt text](https://github.com/vanphan24/cluster-peering-failover-demo/blob/main/images/dashboard-beofre.png)


**This is your current configuration:**  
![alt text](https://github.com/vanphan24/cluster-peering-failover-demo/blob/main/images/diagram-before2.png)


# Deploy Consul on second Kubernetes cluster (dc2).


10. Add Consul Ent license as a K8s secret

```
kubectl create secret generic license --from-literal=key=$CONSUL_LICENSE --context $dc2
```

11. Set context and deploy Consul on dc2

```
kubectl config use-context $dc2
```
```
helm install $dc2 hashicorp/consul --values consul-values-sameness.yaml --set global.datacenter=dc2     
```

or if pointing to pre-reelase K8s chart that is locally clone Consul-k8s repo:
```
helm install $dc2 ../consul-k8s/charts/consul --values consul-values-sameness.yaml --set global.datacenter=dc2                         
```

Note: Run ```kubectl get crd``` and make sure that exportedservices.consul.hashicorp.com exist.    
If not, you need to upgrade your helm deployment:  

```
helm upgrade $dc2 hashicorp/consul --values consul-values-sameness.yaml    
```

12. Deploy counting service on dc2. This will be the failover service instance.

```
kubectl apply -f countingapp/counting.yaml --context $dc2
```


# Create cluster peering connection

You can establish the peering connections using the Consul UI or using Kubernetes CRDs. The steps using the UI are extremely easy and straight forward:

13. Set the Consul servers (control plane) to use mesh gateways when requesting/accepting cluster peering connection. Just apply the meshgw.yaml file on both Kubernetes cluster. 

```
kubectl apply -f meshgw.yaml --context $dc1
kubectl apply -f meshgw.yaml --context $dc2
```

14. Establish cluster peering connecting using the Consul UI of each cluster:

  - Log onto Consul UI for dc1, navigate to the Peers side tab on the left hand side.
  - Click on **Add peer connection***
  - Enter a name you want to represent the peer that you are connecting to. 
  - Click **Generate token**
  - Copy the newly created token.
  - Log onto Consul UI for dc2, navigate to the Peers side tab on the left hand side.
  - Click on **Add peer connection***
  - Click on **Establish peering**
  - Enter a name you want to represent the peer that you are connecting to.
  - Paste the token and click **Add peer**
  - Your peering connection should be established.

15. Apply sameness group configuration entry on each Consul cluster. 
Note : This tells each consul cluster to treat any service that has the same consul namespace and same service name as the same service as long as it is the logical sameness group. The members of the sameness group can include peers or partitions in the members list of the sameness config entry.

```
kubectl apply -f sameness-config-entry.yaml --context $dc1
kubectl apply -f sameness-config-entry.yaml --context $dc2
```

16. Export services from dc2 to dc1.

```
kubectl apply -f exported-service.yaml --context $dc2
```

17. If you have deny-all intentions set or if ACL's are enabled (which means deny-all intentions are enabled), set intentions using intention.yaml file.  

Note: The UI on Consul version 1.14 does not yet recognize peers for Intention creation. Therefore apply intentions using the CLI, API, or CRDs.

```
kubectl apply -f intentions-sameness.yaml --context $dc2
```

18. Apply the proxy-defaults on both datacenters to ensure data plane traffic between dc1 and dc2 clusters goes via local mesh gateways 
```
kubectl apply -f proxydefault.yaml --context $dc1
kubectl apply -f proxydefault.yaml --context $dc2
```


# Failover counting service

19. Delete the counting service on the dc1 cluster.

```
kubectl delete -f countingapp/counting.yaml --context $dc1
```

20. Observe the dashboard service on your browser. You should notice that the counter has restarted since the dashboard is connecting to different counting service instance.

![alt text](https://github.com/vanphan24/cluster-peering-failover-demo/blob/main/images/dashboard-failover.png)

**This is your current configuration:**  
![alt text](https://github.com/vanphan24/cluster-peering-failover-demo/blob/main/images/Screen%20Shot%202022-09-13%20at%205.13.46%20PM.png "Cluster Peering Demo")


21. Bring counting service on dc1 back up.
```
kubectl apply -f countingapp/counting.yaml --context $dc1
```


22. Observe the dashboard service on your browser. Notice the the dashboard URL shows the counter has restarted again since it automatically fails back to the original service on dc1.


