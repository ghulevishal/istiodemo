# Istio Demo

## Prequisites
- You need the Kubernetes cluster up and running.
- Edit the `/etc/kubernetes/manifests/kube-apiserver.yaml` and add following line in `kube-apiserver.yaml` This will enable alpha features.

```
# Add this line in /etc/kubernetes/manifests/kube-apiserver.yaml
- --runtime-config=admissionregistration.k8s.io/v1alpha1
```

- Restart the `kubelet` service and verify the `kubelet` service is running.
```
$ systemctl restart kubelet.service
$ systemctl status kubelet.service
```


## Download the Istio and Install the Istio.
```
$ curl -L https://git.io/getLatestIstio | sh -
```

- Copy the `istioctl` repository to the path.
```
$ cd istio-0.3.0/
$ ls
bin  install  istio.VERSION  LICENSE  README.md  samples

$ sudo cp bin/istioctl /usr/local/bin/.
```

- Update the `istio-ingress service` as `NodePort` in `install/kubernetes/istio.yaml` as shown below. 
```
apiVersion: v1
kind: Service
metadata:
  name: istio-ingress
  namespace: istio-system
  labels:
    istio: ingress
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 32000
    name: http
  - port: 443
    name: https
  selector:
    istio: ingress
```


- Install the Istio in kubernetes cluster.
```
$ kubectl apply -f install/kubernetes/istio.yaml
```

#### Optional: If your cluster has Kubernetes alpha features enabled, and you wish to enable a automatic injection of sidecar, install the Istio-Initializer:
```
$ kubectl apply -f install/kubernetes/istio-initializer.yaml
```

- Verify the Installation by checking deployed pods.
```
$ kubectl get pods -n istio-system
NAME                                READY     STATUS    RESTARTS   AGE
istio-ca-555bbff478-5mqbq           1/1       Running   0          1m
istio-ingress-57b544fd9c-r24kw      1/1       Running   0          1m
istio-initializer-76b55bcb9-vqzdf   1/1       Running   0          53s
istio-mixer-6c67558b4c-jcn74        3/3       Running   0          1m
istio-pilot-5655764fbb-klvxs        2/2       Running   0          1m
```
Make sure that `istio-pilot-*`, `istio-mixer-*`, `istio-ingress-*`, `istio-ca-*`, and, optionally, `istio-initializer-*` Pods are up and running properly.

- Check desired services are running in the cluster.
```
$ kubectl get svc -n istio-system
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                            AGE
istio-ingress   NodePort    10.106.47.35    <none>        80:32000/TCP,443:31206/TCP                                         4m
istio-mixer     ClusterIP   10.103.83.231   <none>        9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   4m
istio-pilot     ClusterIP   10.99.149.121   <none>        15003/TCP,443/TCP                                                  4m
```
 We have deployed Istio in our kubernetes cluster.
 
 Deploy the simple `BookInfo` application in the kubernetes cluster.
 ```
 
 ```
 
 Check the lists of services and pods running.
 ```
 $ kubectl get po
NAME                             READY     STATUS    RESTARTS   AGE
details-v1-5776b48b4d-466jt      2/2       Running   0          5m
productpage-v1-fd64558b8-lwtvq   2/2       Running   0          5m
ratings-v1-5f8f9f5db4-vcnvg      2/2       Running   0          5m
reviews-v1-c96f558c6-czkn9       2/2       Running   0          5m
reviews-v2-b5d8745bd-tqq4c       2/2       Running   0          5m
reviews-v3-79d8ff97d8-nq9q7      2/2       Running   0          5m

$ kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.98.125.108    <none>        9080/TCP   6m
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    48m
productpage   ClusterIP   10.96.80.171     <none>        9080/TCP   6m
ratings       ClusterIP   10.106.128.50    <none>        9080/TCP   6m
reviews       ClusterIP   10.106.106.197   <none>        9080/TCP   6m
 ```

Access the application. As we are not using any load balancer our ingress is exposed at 32000 port of NodeIP.
- Lets find the Gateway URL.
```
$ export GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')

$ echo $GATEWAY_URL
138.197.15.135:32000
```

- You can access the application at the `138.197.15.135:32000/productpage`
Eachtie you refresh the application you will see the review section is changed. As the 3 different review versions are used so each time reviw filed get changed.

## Configure the routing rules.

### Default route

For example, a simple rule to send 100% of incoming traffic for a “reviews” service to version “v1” can be described using the Rules as follow
```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: defaultroute
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v1
    weight: 100
```
- Deploy this route rule.
```
$ kubectl create -f route1.yaml 
routerule "defaultroute" created
```
Try to access the `BookInfo` application at the `138.197.15.135:32000/productpage` and try to refresh the page you will see traffic from reviews” service version “v1” is allowed so there is no any chage in review section.

- Delete the route rule.
```
$ kubectl delete routerule defaultroute
routerule "defaultroute" deleted
```

### Split traffic between service versions.
For example, the following rule will route 25% of traffic for the “reviews” service to instances with the “v2” tag and the remaining traffic (i.e., 75%) to “v1”.

```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: split
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
    weight: 25
  - labels:
      version: v1
    weight: 75

```
- Deploy this rule.
```
$ kubectl create -f split.yaml 
routerule "split" created
```
- Try to access the `BookInfo` application at the `138.197.15.135:32000/productpage` and You can check rule is applied.

- Delete the rule.
```
$ kubectl delete routerule split
routerule "split" deleted
```

### Injecting faults in the request path

#### Delay 
- The following example will introduce a 5 second delay in 10% of the requests to the “v1” version of the “reviews” microservice.
```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: delay
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v1
  httpFault:
    delay:
      percent: 10
      fixedDelay: 5s

```

- Deploy the rule.
```
$ kubectl create -f delay.yaml 
routerule "delay" created
```
- Try to access the `BookInfo` application at the `138.197.15.135:32000/productpage` and You can check rule is applied.

- Delete the rule.
```
$ kubectl delete routerule delay
routerule "delay" deleted
```
