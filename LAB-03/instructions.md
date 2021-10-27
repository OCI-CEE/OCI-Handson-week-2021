# Set up environment variables
On each start of a Cloud Shell, execute:
```bash
export MY_INITIALS=<your initials>
export ATP_OCID=<ocid of your ATP instance>
## once Ingress is set up: 
export EXTERNAL_IP=$(kubectl --namespace ingress-nginx get services -o jsonpath={.status.loadBalancer.ingress[0].ip} ingress-nginx-controller)
```

[TOC]

# [Cloud shell preparation](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=cloud-shell-setup)
```bash
cd $HOME
git clone https://github.com/CloudTestDrive/helidon-kubernetes.git

cd $HOME
oci db autonomous-database generate-wallet --file Wallet.zip --password 'Pa$$w0rd' --autonomous-database-id "$ATP_OCID"
mkdir -p $HOME/helidon-kubernetes/configurations/stockmanagerconf/Wallet_ATP
cd $HOME/helidon-kubernetes/configurations/stockmanagerconf/Wallet_ATP
unzip $HOME/Wallet.zip
ls -Al $HOME/helidon-kubernetes/configurations/stockmanagerconf/Wallet_ATP
cat tnsnames.ora

cd $HOME/helidon-kubernetes/configurations/stockmanagerconf/conf
nano stockmanager-config.yaml  ## interactive
```

[Setup the department ID](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=cloud-shell-setup#Task4:SettingupyourdepartmentId) in the file, then save the file with `CTRL+X` and confirmation.

```bash
mkdir $HOME/keys
cd $HOME/keys
wget -O step.tar.gz https://dl.step.sm/gh-release/cli/docs-ca-install/v0.17.5/step_linux_0.17.5_amd64.tar.gz && tar -xf step.tar.gz
mv step_*/bin/step .
rm -rf *gz step_*
./step certificate create root.cluster.local root.crt root.key --profile root-ca --no-password --insecure
```


# [Part 1](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs)

## [Helm](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task1:ConfiguretheHelmrepository)
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard
helm repo list
helm repo update
```

## [Kubectl](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task2:Gettingyourclusteraccessdetails)
```bash
mkdir -p $HOME/.kube
```
Access and copy the config from OCI UI, according to the instructions.

```bash
chmod 600 $HOME/.kube/config
kubectl get nodes
kubectl config get-contexts
kubectl config rename-context $(kubectl config get-contexts --no-headers -o name) one
kubectl config get-contexts
```

## [Ingress](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task3astartinganingresscontrollerforacceptingexternaldata)
```bash
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --version 3.38.0 --set rbac.create=true --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-protocol"=TCP --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape"=10Mbps
watch -n 2 kubectl --namespace ingress-nginx get services -o wide ingress-nginx-controller
```
`CTRL+C`
```bash
export EXTERNAL_IP=$(kubectl --namespace ingress-nginx get services -o jsonpath={.status.loadBalancer.ingress[0].ip} ingress-nginx-controller)
```

## [Dashboard](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task3binstallingthekubernetesdashboard)
```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --namespace kube-system --set ingress.enabled=true --set ingress.hosts="{dashboard.kube-system.$EXTERNAL_IP.nip.io}" --version 5.0.0
watch -n 2 helm list --namespace kube-system
```
`CTRL+C`

## [Dashboard user](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task3csettingupthekubernetesdashboarduser)
```bash
cd $HOME/helidon-kubernetes/base-kubernetes
kubectl apply -f dashboard-user.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep dashboard-user | awk '{print $1}')
```
Copy and save the token (the dashboard password) locally.

```bash
echo 'https://dashboard.kube-system.'${EXTERNAL_IP}'.nip.io/#!/login'
```
Open the displayed URL in the browser (click on it in the Cloud Shell console). Use the token to login.


## [Namespace, Ingress and Services](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task4:NamespaceServicesandIngressrules)

### [Namespace](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task4acreatinganamespacetoworkin)
```bash
cd $HOME/helidon-kubernetes/base-kubernetes
bash create-namespace.sh ${MY_INITIALS}-helidon
kubectl get namespace
kubectl get all
kubectl get all -n kube-system
```

### [Services](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task4bcreatingservices)
```bash
bash create-services.sh
kubectl get services
```

### [Ingress](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task4caccessingyourservicesusinganingressrule)
```bash
kubectl get services -n ingress-nginx
$HOME/keys/step certificate create store.$EXTERNAL_IP.nip.io tls-store-$EXTERNAL_IP.crt tls-store-$EXTERNAL_IP.key --profile leaf --not-after 8760h --no-password --insecure --ca $HOME/keys/root.crt --ca-key $HOME/keys/root.key
kubectl create secret tls tls-store --key tls-store-$EXTERNAL_IP.key --cert tls-store-$EXTERNAL_IP.crt
kubectl get ingress
bash set-ingress-ip.sh $EXTERNAL_IP
bash create-ingress-rules.sh
kubectl get ingress
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/store
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/unknowningress
```

## [Secrets and Config Maps](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task5:Secretsconfigmapsexternalconfigurationforyourcontainers)

### [Connection details](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task5aconfiguringthedatabaseconnectiondetails)
```bash
cd $HOME/helidon-kubernetes/configurations/stockmanagerconf
nano databaseConnectionSecret.yaml  ## interactive
```

Configure the connection profile name in the file (according to the instructions), then save the file with `CTRL+X` and confirmation.

```bash
cd $HOME/helidon-kubernetes/base-kubernetes
bash create-secrets.sh
kubectl get secret sm-conf-secure -o yaml
kubectl get secret sm-conf-secure -o jsonpath='{.data.stockmanager-security\.yaml}' | base64 -d -i -
```

### [Config maps](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#task5cconfigmaps)
```bash
cd $HOME/helidon-kubernetes/base-kubernetes
bash create-configmaps.sh
kubectl get configmaps
kubectl get configmap sf-config-map -o=yaml
```

## [Deployment!](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task6:Deployingtheactualmicroservices)
```bash
cd $HOME/helidon-kubernetes
bash deploy.sh
watch -n1 kubectl get all
```
`CTRL+C`

```bash
kubectl logs --follow $(kubectl get pod -l app=storefront -o name)
```
`CTRL+C`

```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel

bash create-test-data.sh $EXTERNAL_IP

curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel

kubectl logs $(kubectl get pod -l app=storefront -o name) --tail=5
kubectl logs $(kubectl get pod -l app=stockmanager -o name) --tail=20
```

### Zipkin
```bash
echo https://store.$EXTERNAL_IP.nip.io/zipkin
```
Open the displayed URL in the browser (click on it in the Cloud Shell console).

```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/minimumChange
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/smmgt/health/ready
```


##	[Config updates](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=kubernetes-base-labs#Task7:Updatingyourexternalconfiguration)
```bash
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/status
kubectl exec -it $(kubectl get pod -l app=storefront -o name) -- /bin/bash
```
Within the container:
```bash
ls /conf
cat /conf/storefront-config.yaml
exit
```

Modify the config in the dashboard, according to the instructions.

```bash
kubectl exec -it $(kubectl get pod -l app=storefront -o name) -- /bin/bash
```
Within the container:
```bash
cat /conf/storefront-config.yaml
exit
```
```bash
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/status
```


--------

# [Part 2](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=health-liveness-readiness)

## [Are containers running?](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=health-liveness-readiness#Task2:Isthecontainerrunning?)
```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
kubectl get pods
kubectl exec -it $(kubectl get pod -l app=storefront -o name) -- /bin/bash -c 'kill -1 1; bash'
```
```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
kubectl get pods
kubectl logs $(kubectl get pod -l app=storefront -o name)
```

## [Liveness](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=health-liveness-readiness#Task3:Liveness)
```bash
cd $HOME/helidon-kubernetes
bash undeploy.sh
nano storefront-deployment.yaml ## interactive
```

Uncomment the liveness probe section and save the file.

```bash
bash deploy.sh
kubectl logs --follow $(kubectl get pod -l app=storefront -o name)
```
`CTRL+C`
```bash
kubectl describe $(kubectl get pod -l app=storefront -o name)

kubectl logs --tail=10 --follow $(kubectl get pod -l app=storefront -o name)
```

> ### In a separate window/tab
Duplicate your browser tab, and in the new Cloud Shell:
```bash
kubectl exec -it $(kubectl get pod -l app=storefront -o name) -- /bin/bash
touch /frozen
```
_Leave the tab open_

```bash
kubectl get pods
kubectl describe $(kubectl get pod -l app=storefront -o name)
```


##	[Readiness](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=health-liveness-readiness#Task4:Readiness)
```bash
cd $HOME/helidon-kubernetes
nano storefront-deployment.yaml ## interactive
```
Uncomment the readiness probe section and save the file.

```bash
cd $HOME/helidon-kubernetes
bash undeploy.sh
kubectl get all
bash deploy.sh
watch -n1 kubectl get all
```
`CTRL+C`

```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
kubectl delete -f stockmanager-deployment.yaml
kubectl get pods
watch -n1 kubectl get all
```
`CTRL+C`

```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
kubectl apply -f stockmanager-deployment.yaml
watch -n1 kubectl get all
```
`CTRL+C`

```bash
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel
```


## [Startup probes](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=health-liveness-readiness#Task5:Startupprobes)
```bash
cd $HOME/helidon-kubernetes
nano storefront-deployment.yaml ## interactive
```
Uncomment the startup probe section and save the file.
```bash
kubectl delete -f storefront-deployment.yaml
kubectl apply -f storefront-deployment.yaml
watch -n1 kubectl get all
```
`CTRL+C`


# [Part 3a](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=horizontal-scaling)

All activities are completed through the Kubernetes dashboard.

# [Part 3b](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=auto-scaling)

##	[Metrics server](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=auto-scaling#Task1:HorizontalautoscalingbasedonCPUloadorMemoryusage)
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install metrics-server bitnami/metrics-server --namespace kube-system --set apiService.create=true
watch -n1 kubectl get deployments -n kube-system
```
`CTRL+C`

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n kube-system
```

> ### In a separate window/tab
Duplicate your browser tab, and in the new Cloud Shell:
```bash
export EXTERNAL_IP=$(kubectl --namespace ingress-nginx get services -o jsonpath={.status.loadBalancer.ingress[0].ip} ingress-nginx-controller)
cd $HOME/helidon-kubernetes/cloud-native-kubernetes/auto-scaling
bash generate-load.sh $EXTERNAL_IP 0.1
```

```bash
watch -n5 kubectl top pods
```
`CTRL+C`

## [Autoscaler](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=auto-scaling#Task2:Configuringtheautoscaler)
```bash
kubectl autoscale deployment storefront --min=2 --max=5 --cpu-percent=25
kubectl get horizontalpodautoscaler storefront
kubectl describe hpa storefront

watch -n5 kubectl get hpa storefront
```
`CTRL+C`

```bash
kubectl delete hpa storefront
kubectl scale --replicas=1 deployment storefront
watch -n1 kubectl get all
```
`CTRL+C`


# [Part 4](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=rolling-updates)

## [Rolling updates](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=rolling-updates#Task2:Howtodoarollingupgradeinoursetup)
```bash
cd $HOME/helidon-kubernetes
cp storefront-deployment.yaml storefront-deployment-v0.0.1.yaml
nano storefront-deployment-v0.0.1.yaml ## interactive
```
Configure the number of replicas and update strategy, save the file.

```bash
kubectl apply -f storefront-deployment-v0.0.1.yaml --record
kubectl rollout status deployment storefront
kubectl rollout history deployment storefront
```

## [Changing the pods](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=rolling-updates#makingachangethatupdatesthepods)
```bash
kubectl set image deployment storefront storefront=fra.ocir.io/oractdemeabdmnative/h-k8s_repo/storefront:0.0.2 --record
watch -n1 "curl -s -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/status && echo && kubectl get all"
```
`CTRL+C`

```bash
kubectl rollout status deployment storefront
kubectl get all
kubectl rollout history deployment storefront
kubectl describe deployment storefront
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/status && echo
curl -i -k -X GET -u jack:password https://store.$EXTERNAL_IP.nip.io/store/stocklevel && echo
```

## [Rolling back updates](https://oracle.github.io/cloudtestdrive/AppDev/cloud-native/livelabs/individual/kubernetes/kubernetes-core/index.html?lab=rolling-updates#Task3:Rollingbackaupdate)
```bash
kubectl get replicasets
kubectl describe replicaset $(kubectl get replicaset -l app=storefront -o jsonpath='{.items[?(@.status.replicas > 0)].metadata.name}')
kubectl describe replicaset $(kubectl get replicaset -l app=storefront -o jsonpath='{.items[?(@.status.replicas == 0)].metadata.name}')
kubectl rollout undo deployment storefront
kubectl rollout status deployment storefront
kubectl get all
curl -i -k -X GET https://store.$EXTERNAL_IP.nip.io/sf/status && echo
```
