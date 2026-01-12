Neuvector在部署时有多种方式，比如docker、docker-compose、kubernetes、helm、openshift等。

# 一、组件升级

当有一个新的版本可用时，从DockerHub仓库拉取即可。

**滚动升级**

**滚动升级**

**滚动升级**

建议采用“滚动升级”策略，保证至少有一个AllinOne容器或控制器容器在升级期间可以使用。



## 1、2 helm部署

如果采用helm chart来升级，升级操作将处理额外的services、rolebindings以及其他升级要求。



## 1、3 策略和配置持久化

如果手动更新或者只有一个 Allinone 或 Controller 在运行，此时当前的网络连接数据不会被存储，并且会在 NeuVector 容器停止时丢失。



Neuvector支持策略和配置的持久化。它配置实时数据备份到/var/neuvector挂载卷上。

当挂载pv后，配置和策略会在运行时存储到pv。

在集群完全失败的情况下，配置会在新集群创建时自动恢复。

配置和策略也可以从 /var/neuvector/ 卷中手动恢复或删除。



## 使用docker-compose手动更新

```
docker-compose -f <filename> down
```



image修改成想要升级的镜像版本

```
docker-compose -f <filename> up -d
```

备注：建议同时将所有 NeuVector 组件更新到最新版本。



# 二、滚动更新

K8s，openshift，Rancher都支持滚动更新。可使用此特性来更新Neuvector容器，确保至少有一个Allinone或控制器在运行，策略、日志和连接数据才不会丢。确保容器更新间至少有 30 秒的间隔，以便可以选举新的leader并在控制器之间同步数据。

## 2、1 k8s滚动升级

编辑yaml文件更改镜像地址，然后apply应用

```
kubectl apply -f <yaml file>
```



命令行来升级新版本的Neuvector

```
kubectl set image deployment/neuvector-controller-pod neuvector-controller-pod=neuvector/controller:4.2.2 -n neuvector
kubectl set image deployment/neuvector-manager-pod neuvector-manager-pod=neuvector/manager:4.2.2 -n neuvector
kubectl set image DaemonSet/neuvector-enforcer-pod neuvector-enforcer-pod=neuvector/enforcer:4.2.2 -n neuvector
```

检测滚动升级的状态

```
kubectl rollout status -n neuvector ds/neuvector-enforcer pod
kubectl rollout status -n neuvector deployment/neuvector-controller-pod  // same for manager, scanner etc
```

回滚更新

```
kubectl rollout undo -n neuvector ds/neuvector-enforcer-pod
kubectl rollout undo -n neuvector deployment/neuvector-controller-pod   // same for manager, scanner etc
```



# 三 升级CVE数据库

默认的 NeuVector 部署包括Scanner pod 的部署以及 Updater cronjob去每天更新Scanner 。

## 3、1 查看cve数据库版本

您还可以检查Updater 容器映像。

```
docker inspect neuvector/updater
```

```
"Labels": {
                "neuvector.image": "neuvector/updater",
                "neuvector.role": "updater",
                "neuvector.vuln_db": "1.255"
            }
```



您还可以检查controller/allinone 日志中的“版本”。

```
kubectl logs neuvector-controller-pod-777fdc5668-4jkjn -n neuvector | grep version
```

```
DEBU|SCN|main.dbUpdate: New DB found - create=2019-07-24T11:59:13Z version=1.576
DEBU|SCN|memdb.ReadCveDb: New DB found - update=2019-07-24T11:59:13Z version=1.576
DEBU|SCN|main.scannerRegister: - version=1.576
```



检查docker logs日志

```
docker logs <scanner container id or name> | grep version
```

```
2020-09-15T00:00:57.909|DEBU|SCN|memdb.ReadCveDb: New DB found - update=2020-09-14T10:37:56Z version=2.04
2020-09-15T00:01:10.06 |DEBU|SCN|main.scannerRegister: - entries=47016 join=neuvector-svc-controller.neuvector:18400 version=2.040
```



## 3、2 更新CVE数据库

默认情况下，Updater Cron Job会自动检查新的CVE数据库更新，通过DockerHub上发布的新版本的scanner。

对于手动从 NeuVector 拉取镜像的部署方式，应拉取带有“latest”标签的scanner以更新 CVE 数据库。



对于镜像仓库扫描，如果启用了“CVE DB 更新后重新扫描”选项，则该仓库中的所有镜像将在 CVE 数据库更新后重新扫描。

### 3、2、1 Updater Cron Job

每天午夜运行更新程序，

neuvector-updater.yaml文件内容如下：

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-updater-pod
  namespace: neuvector
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: neuvector-updater-pod
        spec:
          containers:
          - name: neuvector-updater-pod
            image: neuvector/updater
            imagePullPolicy: Always
            lifecycle:
              postStart:
                exec:
                  command:
                  - /bin/sh
                  - -c
                  - TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`; /usr/bin/curl -kv -X PATCH -H "Authorization:Bearer $TOKEN" -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'`date +%Y-%m-%dT%H:%M:%S%z`'"}}}}}' 'https://kubernetes.default/apis/apps/v1/namespaces/neuvector/deployments/neuvector-scanner-pod'
            env:
              - name: CLUSTER_JOIN_ADDR
                value: neuvector-svc-controller.neuvector
          restartPolicy: Never
```

**注意：**如果部署了 allinone 容器而不是controller控制器，请将 neuvector-svc-controller.neuvector 替换为 neuvector-svc-allinone.neuvector



运行Cron Job任务

### 3、2、2 Docker原生升级

备注：拉取和运行Scanner映像时始终使用 :latest 标签，以确保部署最新的 CVE 数据库。

**对docker而言**

```
docker stop scanner
docker rm <scanner id>
docker pull neuvector/scanner:latest
<docker run command from below>
```



**对于docker-compose**

```
docker-compose -f file.yaml down
docker-compose -f file.yaml pull        // pre-pull the image before starting the scanner
docker-compose -f file.yaml up -d
```

### 3、2、3 Kubernetes平台手动更新

运行以下的updater文件

```
kubectl create -f neuvector-manual-updater.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: neuvector-updater-pod
  namespace: neuvector
spec:
  containers:
  - name: neuvector-updater-pod
    image: neuvector/updater
    imagePullPolicy: Always
    lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - -c
          - TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`; /usr/bin/curl -kv -X PATCH -H "Authorization:Bearer $TOKEN" -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'`date +%Y-%m-%dT%H:%M:%S%z`'"}}}}}' 'https://kubernetes.default/apis/apps/v1/namespaces/neuvector/deployments/neuvector-scanner-pod'
    env:
      - name: CLUSTER_JOIN_ADDR
        value: neuvector-svc-controller.neuvector
  restartPolicy: Never
```

# 四、scanner原理分析

将CVE数据库扣出来，

kubeadm更换仓库

https://hub.docker.com/r/labring/kubernetes/tags
