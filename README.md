# Istiodemo

## Start the Minikube.
```
$ sudo minikube start --memory 4096 --feature-gates=Accelerators=true --extra config=apiserver.Admission.PluginNames="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,GenericAdmissionWebhook,ResourceQuota"
 
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
