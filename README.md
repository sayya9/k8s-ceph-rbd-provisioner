### **前置準備**
---

於ceph admin node登入，部署ceph client(也就是k8s node)：

```
$ cd my-cluster
$ ceph-deploy install k8s-node1
$ ceph-deploy admin k8s-node1
```

於ceph client(k8s node)登入，建立一個kube pool(此步驟只限首次，如果只是部署ceph client則不需要做)
```
$ ceph osd pool create kube 8 8
```


建立admin以及user(命名為```kube```) key
* admin key：ceph預設就存在admin key，故不需要額外建立
```
$ ceph auth get client.admin 2>&1 | grep "key = " | awk '{print  $3}' | xargs echo -n
```

* user key：記得命名為kube
```
$ ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
$ ceph auth get client.kube 2>&1 | grep "key = " | awk '{print  $3}' | xargs echo -n
```

### **安裝篇**
---

下載 ceph-rbd-provisioner helm配置檔
```
$ git clone https://github.com/sayya9/k8s-ceph-rbd-provisioner.git
$ cd k8s-ceph-rbd-provisioner/helm/ceph-rbd-provisioner
```

開始安裝ceph-rbd-provisioner
* 把上面步驟撈取的admin、user key放進去value.yaml的adminKey與userKey。其實這兩把key會使用base64編碼，再放入templates/{ceph-admin-secret.yaml,ceph-user-secret.yaml} (type是 Opaque，Opaque就是base64編碼格式)

```
$ cat values.yaml | grep Key
  adminKey: AQBP2fVZWiVfEBAA3i9wcPGP9QbgoPXDqVxH9Q==
  userKey: AQA82/VZTA1tFRAAhVkTVZQteRG035qExacIpQ==
```

* 告知ceph monitor在哪
```
cat values.yaml | grep mon
  mon: 192.168.57.101:6789
```

* 開始安裝了
```
$ helm install .
```

### **測試篇**
產生一個pvc來測試
```
$ cat ~/test-rbd-pvc.yaml
kind: "PersistentVolumeClaim"
apiVersion: "v1"
metadata:
  name: "test-rbd-pvc"
  annotations:
    volume.beta.kubernetes.io/storage-class: "rbd"
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "1Gi"

$ kubectl apply -f ~/test-rbd-pvc.yaml
$ kubectl get pvc
NAME           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-rbd-pvc   Bound     pvc-7944d246-bcb4-11e7-88fa-525400ada096   1Gi        RWO            rbd            1m
```

產生一個pod來掛載它
```
$ cat ~/mypod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: test-rbd-pvc
        
$ kubectl apply -f ~/mypod.yaml
$ kubectl exec -it mypod bash
```

於pod內查看
```
root@mypod:/# df -h /var/www/html/
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0       976M  2.6M  907M   1% /var/www/html
```

參考：
* https://docs.oracle.com/cd/E52668_01/E66514/html/ceph-cephfS.html
* http://docs.ceph.com/docs/kraken/cephfs/createfs/
* https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd/examples
* https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd/deploy/rbac
