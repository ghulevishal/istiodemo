# Istiodemo

## Start the Minikube.
```
$ minikube start --memory 4096 --feature-gates=Accelerators=true --extra config=apiserver.Admission.PluginNames="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,GenericAdmissionWebhook,ResourceQuota"
  
```
