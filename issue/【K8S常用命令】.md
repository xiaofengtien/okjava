### 查看运行实例

```bash
kubectl get pods -n test |grep wms 查看服务器运行实例
```

### 查看日志

```bash
kubectl logs -f wms-center-596b8579bc-9vwjl -n dev 具体应用日志
查询日志
kubectl logs -f  $(kubectl get pods -n test |grep wms-center |awk '{print $1}') -n test |grep 'dd'
kubectl logs -f --tail=2000 $(kubectl get pods -n test |grep wms-strategy |awk '{print $1}') -n test 
kubectl logs -f --tail=2000 $(kubectl get pods -n test |grep wms-admin |awk '{print $1}') -n test 
kubectl logs -f --tail=20000 $(kubectl get pods -n test |grep wms-app |awk '{print $1}') -n test 
kubectl logs -f --tail=2000 $(kubectl get pods -n test |grep wms-inventory |awk '{print $1}') -n test 
kubectl logs -f    $(kubectl get pods -n test |grep wms-delivery |awk '{print $1}') -n test |grep 'DeliveryOrderServiceImpl:createFromUpstream'
kubectl logs -f --tail=2000 $(kubectl get pods -n dev |grep wms-app |awk '{print $1}') -n dev 
kubectl logs -f --tail=2000 $(kubectl get pods -n dev |grep wms-delivery |awk '{print $1}') -n dev 
kubectl logs -f --tail=2000 $(kubectl get pods -n dev |grep wms-center |awk '{print $1}') -n dev 
kubectl logs -f --tail=2000 $(kubectl get pods -n dev |grep wms-admin |awk '{print $1}') -n dev
```

### 查看dockfile配置

``` bash
kubectl get pod wms-admin-664f656ff-h8v8n -n test -o yaml   
```

### 进入容器内部

```bash
kubectl exec -it <pod_name> -n test sh
```

