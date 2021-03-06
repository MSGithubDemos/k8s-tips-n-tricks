# Kubernetes tips and tricks

## kubectl commands

### Official kubectl Cheat Sheet
https://kubernetes.io/docs/reference/kubectl/cheatsheet/

### View resource usage
```
kubectl top no
kubectl top po
```
### Display docker image tage and SHA
`kubectl get pod <my-pod-name> -o json | jq '.status.containerStatuses[] | { "image": .image, "imageID": .imageID }'`

### Display logs for previous started container to debug abnormal successive restarts
`kubectl logs <my-pod> --previous`

### Display http requests made by kubectl to kube-api 
`kubectl get po -v=6`

### Extract a token from a secret
`kubectl get secret <my-secret> -n<my-namespace> -ojsonpath='{.data.token}' | base64 -d`

### Extract the content of a file from a secret (dot in file name must be escaped by \\)
`kubectl get secret <my-secret> -ojsonpath='{.data.jmxremote\\.password}' | base64 -d`

### Copy an object from one namespace to another
`kubectl get secrets <my-secret> -ojson -n<my-src-namespace> | jq '.metadata.namespace = "<my-dest-namespace>"' | kubectl create -f -`

### Clean up an helm release manually
`kubectl get deploy,sts,configmaps,secret,pvc,svc -oname -l release=<my-helm-release> | while read name; do kubectl delete $name; done`

### Wait for pod to be ready
`kubectl wait po <my-po> --for=condition=Ready`

### Watch pods
`watch kubectl get po -lrelease=<my-helm-release>`

### Find pods by date with jq
```
DEPLOYMENT_STARTDATE=`jq -n 'now'`
kubectl get po -lrelease=<my-helm-release> -ojson | jq -r --arg deployment_startdate $DEPLOYMENT_STARTDATE '.items[] | select(.metadata.creationTimestamp | fromdate | tostring > $deployment_startdate) | .metadata.name'
```

### Find pods using a specific environment variable in secret
```
kubectl get pods -o json | jq -r '.items[] | select(.spec.containers[].env[]?.valueFrom.secretKeyRef.key=="<MY_VAR_ENV_NAME>") | .metadata.name'
```

### Inject an environment variable in a deployment
`kubectl set env deployment/registry STORAGE_DIR=/local`

### Restart pods properly with a rollout (from 1.15)
`kubectl rollout restart deploy <my-deploy>`

### Check if I'm allowed to do an action
`kubectl auth can-i exec pod`

### Suspend all cronjobs at once
`kubectl get cronjobs.batch -oname | while read name; do kubectl patch $name -p '{"spec":{"suspend":true}}'; done`

### List image in deployments
`kubectl get deploy -lrelease=si-labo -ojson | jq .items[].spec.template.spec.containers[0].image`

### List pods in status other than Running or Completed
`kubectl get po -owide --all-namespaces | grep -v 'Running\|Completed'`

### List evicted pods on all cluster
- `kubectl get pods --field-selector=status.phase=Failed --all-namespaces -owide`
- `kubectl get pods --all-namespaces -ojson | jq -r '.items[] | select(.status.reason=="Evicted") | .metadata.namespace + " " + .spec.nodeName + " " + (.spec.priority|tostring)+ " " + .metadata.name  + " : " + .status.message' | sort -k2,2 -k3nr`

### List pods with anti affinity 
`kubectl get pods -ojson | jq '.items[] | select(.spec.affinity.podAntiAffinity!=null) |  .metadata.name'`

### List pods with a guaranteed qos
`kubectl get pods -ojson  | jq '.items[] | select(.status.qosClass=="Guaranteed") |  .metadata.name'`

### List prority classes sort by value
`kubectl get priorityclasses -ojson | jq -r '.items[] | .metadata.name + " : " + (.value|tostring)' | sort -k3nr`

### List priority infos for all pods sort by value
`kubectl get pods -ojson | jq -r '.items[] | .metadata.namespace + " : " + .spec.nodeName + " : " + .metadata.name + " : " + .spec.priorityClassName+ " : " + (.spec.priority|tostring)' | sort -k9nr -k5`

### List pods by restart count
`kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'`

### List OOMKilled pods
`kubectl get po --all-namespaces -ojson | jq -r '.items[] | select(.status.containerStatuses[0].lastState.terminated.reason=="OOMKilled") | .metadata.namespace + " " + (.status.containerStatuses[0].restartCount|tostring) + " " + .metadata.name' | sort -k1,1r -k2nr`

### Force delete of pods stuck in terminating status
`kubectl delete pods <pod> --grace-period=0 --force`

### List pre hook jobs for a release
`kubectl get jobs -ojson | jq -r '.items[] | select(.metadata.annotations["helm.sh/hook"] and (.metadata.annotations["helm.sh/hook"]|contains("pre")) and .metadata.labels.release=="<my-helm-release>") | .metadata.name'`

### List pod owned by a hook job
`kubectl get po -ojson | jq -r '.items[] | select(.metadata.ownerReferences[].name == "<my-hook-name>") | .metadata.name'`

### Delete succeeded jobs
`kubectl get jobs -ojson | jq -r '.items[] | select(.metadata.annotations["helm.sh/hook"] and .status.succeeded==1) | .metadata.name' | while read name; do kubectl delete jobs $name ; done`

### List nodes with memory or disk pressure Taint Based Evictions
`kubectl get no -ojson | jq -r '.items[] | select(.spec.taints!=null and (.spec.taints[0].key|contains("pressure"))) | .metadata.name + " : " + .spec.taints[0].key'`

### List allocated ressources per node
`kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c 'echo {}; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve -- ; echo'`
`for i in {01..12}; do echo dbk-k8s-worker-dev-${i}v; kubectl describe node dbk-k8s-worker-dev-${i}v|grep -A6 'Allocated resources:'; done`

### Drain a node
`kubectl drain <node> --ignore-daemonsets --force --delete-local-data`

## Some recipes

### Browse google registry
https://console.cloud.google.com/gcr/images/google-containers/GLOBAL

### Ease kubectl use
![Test](/images/kubectl-tools.jpg?raw=true)

### Decode all secret content easily with ksd
https://github.com/ashleyschuett/kubernetes-secret-decode

`kubectl get secret my-secret -o yaml | ksd`

### Add fuzzy search to your command with fzf
https://github.com/junegunn/fzf

`kubectl get po | fzf`

### Read logs from all replicas at a time with stern
https://github.com/wercker/stern

### Advanced Kubernetes Objects You Need to Know
https://engineering.opsgenie.com/advanced-kubernetes-objects-53f5e9bc0c28

### Interact with kube-api like any other API
https://thenewstack.io/taking-kubernetes-api-spin/

### How to terminate a side-car container in Kubernetes Job
https://medium.com/@cotton_ori/how-to-terminate-a-side-car-container-in-kubernetes-job-2468f435ca99

### Docker Awareness in Java
- https://efekahraman.github.io/2018/04/docker-awareness-in-java
- https://blog.csanchez.org/2017/05/31/running-a-jvm-in-a-container-without-getting-killed/
- https://blogs.oracle.com/java-platform-group/java-se-support-for-docker-cpu-and-memory-limits
- https://banzaicloud.com/blog/java-resource-limits/

### Docker-in-Docker on Kubernetes
https://applatix.com/case-docker-docker-kubernetes-part-2/

### How To Back Up and Restore a Kubernetes Cluster using Heptio Ark
https://www.digitalocean.com/community/tutorials/how-to-back-up-and-restore-a-kubernetes-cluster-on-digitalocean-using-heptio-ark

### Configuring the Kubernetes CLI by using service account tokens
https://www.ibm.com/developerworks/community/blogs/fe25b4ef-ea6a-4d86-a629-6f87ccf4649e/entry/Configuring_the_Kubernetes_CLI_by_using_service_account_tokens1?lang=en

### Treat your pods according to their needs - three QoS classes in Kubernetes
https://cloudowski.com/articles/three-qos-classes-in-kubernetes/

### Prometheus Operator
https://github.com/helm/charts/tree/master/stable/prometheus-operator
https://github.com/coreos/prometheus-operator/tree/master/Documentation
https://sysdig.com/blog/kubernetes-monitoring-prometheus-operator-part3/

### Kube eagle: prometheus exporter and grafana dashboard for a nice overview of a cluster
https://github.com/cloudworkz/kube-eagle

## Resources
- https://hackernoon.com/top-10-kubernetes-tips-and-tricks-27528c2d0222
- https://github.com/mhausenblas/kubectl-in-action
- https://discuss.kubernetes.io/t/kubectl-tips-and-tricks/
