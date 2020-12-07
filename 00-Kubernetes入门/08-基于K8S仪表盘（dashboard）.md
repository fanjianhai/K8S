# 1. 前情回顾

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201119173201200.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

# 2. K8S的GUI资源管理插件--仪表盘

## 2.1. 部署Kubernetes-dashboard

https://github.com/kubernetes/dashboard

### 准备dashboard镜像

```bash
[root@hdss7-200 ~]# docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3

v1.8.3: Pulling from k8scn/kubernetes-dashboard-amd64
a4026007c47e: Pull complete 
Digest: sha256:ebc993303f8a42c301592639770bd1944d80c88be8036e2d4d0aa116148264ff
Status: Downloaded newer image for k8scn/kubernetes-dashboard-amd64:v1.8.3
docker.io/k8scn/kubernetes-dashboard-amd64:v1.8.3
[root@hdss7-200 ~]# 
[root@hdss7-200 ~]# docker images | grep dashboard
k8scn/kubernetes-dashboard-amd64   v1.8.3                     fcac9aa03fd6        2 years ago         102MB
[root@hdss7-200 ~]# docker tag fcac9aa03fd6 harbor.od.com/public/dashboard:v1.8.3
[root@hdss7-200 ~]# docker login harbor.od.com
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@hdss7-200 ~]# docker push harbor.od.com/public/dashboard:v1.8.3
The push refers to repository [harbor.od.com/public/dashboard]
23ddb8cbb75a: Pushed 
v1.8.3: digest: sha256:ebc993303f8a42c301592639770bd1944d80c88be8036e2d4d0aa116148264ff size: 529
```

### 准备资源配置清单

https://github.com/kubernetes/kubernetes/tree/release-1.8/cluster/addons/dashboard

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020111918263050.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

`在hdss7-200.host.com上`

- rbac.yaml

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
```

- dp.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: kubernetes-dashboard
        image: harbor.od.com/public/dashboard:v1.8.3
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard-admin
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
```

- svc.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 443
    targetPort: 8443
```

- ingress.yaml

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: dashboard.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

### 应用资源清单配置

```bash
[root@hdss7-22 ~]# 
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/rbac.yaml
serviceaccount/kubernetes-dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/dp.yaml
deployment.apps/kubernetes-dashboard created
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/svc.yaml
service/kubernetes-dashboard created
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/ingress.yaml
ingress.extensions/kubernetes-dashboard created
[root@hdss7-22 ~]# kubectl get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-6b6c4f9648-pbrgt                1/1     Running   1          39h
kubernetes-dashboard-76dcdb4677-tvwgt   1/1     Running   0          4m8s
traefik-ingress-hndqx                   1/1     Running   1          17h
traefik-ingress-x5cdk                   1/1     Running   1          17h

[root@hdss7-22 ~]# kubectl get svc -n kube-system
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
coredns                   ClusterIP   192.168.0.2      <none>        53/UDP,53/TCP,9153/TCP   39h
kubernetes-dashboard      ClusterIP   192.168.42.9     <none>        443/TCP                  4m22s
traefik-ingress-service   ClusterIP   192.168.159.92   <none>        80/TCP,8080/TCP          18h
[root@hdss7-22 ~]# kubectl get ingress -n kube-system
NAME                   HOSTS              ADDRESS   PORTS   AGE
kubernetes-dashboard   dashboard.od.com             80      4m38s
traefik-web-ui         traefik.od.com               80      18h

```

### 配置域名

`在hdss7-11.host.com上`

```bash
[root@hdss7-11 ~]# cat /var/named/od.com.zone 
$ORIGIN od.com.
$TTL 600    ; 10 minutes
@           IN SOA  dns.od.com. dnsadmin.od.com. (
                2020111305 ; ser在·ial
                10800      ; refresh (3 hours)
                900        ; retry (15 minutes)
                604800     ; expire (1 week)
                86400      ; minimum (1 day)
                )
                NS   dns.od.com.
$TTL 60 ; 1 minute
dns                A    10.4.7.11
harbor		   A    10.4.7.200
k8s-yaml	   A    10.4.7.200
traefik		   A    10.4.7.10
dashboard	   A    10.4.7.10


# 重启dns
[root@hdss7-11 ~]# systemctl restart named

# 测试
[root@hdss7-11 ~]# dig -t A dashboard.od.com @10.4.7.11 +short
10.4.7.10

# coredns的svc
[root@hdss7-21 ~]# dig -t A dashboard.od.com @192.168.0.2 +short
10.4.7.10

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201120095902383.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

### 配置证书

`在主机hdss7-11.host.com上`

```bash
[root@hdss7-11 ~]# cd /etc/nginx/
[root@hdss7-11 nginx]# mkdir certs

# 拷贝证书
[root@hdss7-11 nginx]# cd certs/
[root@hdss7-11 certs]# scp hdss7-200.host.com:/opt/certs/dashboard.od.com.key .
The authenticity of host 'hdss7-200.host.com (10.4.7.200)' can't be established.
ECDSA key fingerprint is SHA256:5ER7p/tIoCgVg4GVnLGuipfiTYiZZmWlDD1EuzkkM04.
ECDSA key fingerprint is MD5:8a:ef:9d:a8:e2:a9:35:c9:1e:10:d5:bf:7a:28:de:14.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'hdss7-200.host.com,10.4.7.200' (ECDSA) to the list of known hosts.
root@hdss7-200.host.com's password: 
dashboard.od.com.key                                                     100% 1675     1.7MB/s   00:00    
[root@hdss7-11 certs]# scp hdss7-200.host.com:/opt/certs/dashboard.od.com.crt .
root@hdss7-200.host.com's password: 
dashboard.od.com.crt                                                     100% 1196     1.4MB/s   00:00  

# 修改nginx配置文件
# vi /etc/nginx/conf.d/dashboard.od.com.conf
server {
  listen        80;
  server_name   dashboard.od.com;
  rewrite ^(.*)$ https://${server_name}$1 permanent;
}
server {
  listen        443 ssl;
  server_name   dashboard.od.com;

  ssl_certificate "certs/dashboard.od.com.crt";
  ssl_certificate_key "certs/dashboard.od.com.key";
  ssl_session_cache shared:SSL:1m;
  ssl_session_timeout  10m;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;


  location / {
    proxy_pass http://default_backend_traefik;
    proxy_set_header Host	$http_host;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
  }
}
# nginx -t
# nginx -s reload

# 获取token
[root@hdss7-21 conf]# kubectl get secret -n kube-system
NAME                                     TYPE                                  DATA   AGE
coredns-token-5h2tb                      kubernetes.io/service-account-token   3      2d
default-token-z5bgv                      kubernetes.io/service-account-token   3      7d13h
kubernetes-dashboard-admin-token-tm2fx   kubernetes.io/service-account-token   3      9h
kubernetes-dashboard-key-holder          Opaque                                2      9h
traefik-ingress-controller-token-dpvlz   kubernetes.io/service-account-token   3      27h
[root@hdss7-21 conf]# kubectl describe kubernetes-dashboard-admin-token-tm2fx -n kube-system
error: the server doesn't have a resource type "kubernetes-dashboard-admin-token-tm2fx"
[root@hdss7-21 conf]# kubectl describe secret kubernetes-dashboard-admin-token-tm2fx -n kube-system
Name:         kubernetes-dashboard-admin-token-tm2fx
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: de80b0e5-803f-4364-b462-6d907e28cb04

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1346 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi10bTJmeCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImRlODBiMGU1LTgwM2YtNDM2NC1iNDYyLTZkOTA3ZTI4Y2IwNCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.cH3OMLinF5odibGFDIvov65eTPjI2I1_NSr0kZMmBtj1mZg8IFumSpdK50j1T6j9hz7MOARVZedP-J10sKFgeR5K7hioPv3UnX1MCnerdpTHDkpnZi4yjOIlhODXevuAa0REN71KpDU0FGXtcCHZuSNG1yuxiD5p3pCCJV9mBCzdRo2070R4WBpC1eoxWjOVpMGPXt8hySl_4sMV1VC6H-1jUZnlDfKLqVW-LodRlUjhUHjln9agM7hxYI3Z6RsuYfqpHGvTuBN7lpkD7ec8RhlYqzKQlTFVNMr4BfZJ0cjVY8adftAC4n4y__zZamzA-3Is4gQ7AepsalT2Kmewkg
```

- 输入token登录（新版强制输入token）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201120190419544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

- 下载新版dashboard

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201120190510717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)



## 2.2. RBAC鉴权原理详解

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020112010105268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

- 用户账户 kubelet 的配置文件中kubelet.kubeconfig

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201120142951632.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

- 所有的Pod都有一个服务账户

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
```

```bash
# 查询角色
[root@hdss7-21 ~]# kubectl get clusterrole
# 查询指定角色
[root@hdss7-21 ~]# kubectl get clusterrole cluster-admin -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-11-12T20:29:51Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "40"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/cluster-admin
  uid: a890e0da-500f-4c5e-aaf8-a37a3d4e4592
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'

```

### 官方github dashboard minimal版本的yaml

https://github.com/kubernetes/kubernetes/tree/release-1.8/cluster/addons/dashboard

```bash
#######################################
# rbac-minimal.yaml
#######################################
[root@hdss7-200 dashboard]# cat rbac-minimal.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
  
  
#######################################
# dp-minimal.yaml
#######################################
[root@hdss7-200 dashboard]# cat dp-minimal.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: kubernetes-dashboard
        image: harbor.od.com/public/dashboard:v1.8.3
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"

```

- 应用资源

```bash
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/rbac-minimal.yaml
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
[root@hdss7-22 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/dp-minimal.yaml
deployment.apps/kubernetes-dashboard configured

# 滚动升级
[root@hdss7-22 ~]# kubectl get pods -n kube-system
NAME                                    READY   STATUS              RESTARTS   AGE
coredns-6b6c4f9648-pbrgt                1/1     Running             1          2d15h
kubernetes-dashboard-6d58ccc9fc-bvs8b   0/1     ContainerCreating   0          2s
kubernetes-dashboard-76dcdb4677-28jf7   1/1     Running             0          3m1s
traefik-ingress-hndqx                   1/1     Running             1          42h
traefik-ingress-x5cdk                   1/1     Running             1          42h
[root@hdss7-22 ~]# kubectl get pods -n kube-system
NAME                                    READY   STATUS        RESTARTS   AGE
coredns-6b6c4f9648-pbrgt                1/1     Running       1          2d15h
kubernetes-dashboard-6d58ccc9fc-bvs8b   1/1     Running       0          4s
kubernetes-dashboard-76dcdb4677-28jf7   0/1     Terminating   0          3m3s
traefik-ingress-hndqx                   1/1     Running       1          42h
traefik-ingress-x5cdk                   1/1     Running       1          42h
[root@hdss7-22 ~]# kubectl get pods -n kube-system
NAME                                    READY   STATUS        RESTARTS   AGE
coredns-6b6c4f9648-pbrgt                1/1     Running       1          2d15h
kubernetes-dashboard-6d58ccc9fc-bvs8b   1/1     Running       0          14s
kubernetes-dashboard-76dcdb4677-28jf7   0/1     Terminating   0          3m13s
traefik-ingress-hndqx                   1/1     Running       1          42h
traefik-ingress-x5cdk                   1/1     Running       1          42h
[root@hdss7-22 ~]# kubectl get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-6b6c4f9648-pbrgt                1/1     Running   1          2d15h
kubernetes-dashboard-6d58ccc9fc-bvs8b   1/1     Running   0          24s
traefik-ingress-hndqx                   1/1     Running   1          42h
traefik-ingress-x5cdk                   1/1     Running   1          42h

```

- 对minimal权限不够

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201121101205143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

### 对yaml文件的详解

```bash
####################
# treafik  rbac.yaml
####################
[root@hdss7-200 traefik]# cat rbac.yaml 
# 创建服务账户traefik-ingress-controller
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
# 创建集群角色traefik-ingress-controller
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
# 创建集群角色绑定traefik-ingress-controller，并让服务账户具有相应的权限
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system


####################
# traefic  ds.yaml
####################
[root@hdss7-200 traefik]# cat ds.yaml 
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: harbor.od.com/public/traefik:v1.7.2-alpine
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://10.4.7.10:7443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus

####################
# traefic  svc.yaml
####################
[root@hdss7-200 traefik]# cat svc.yaml 
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress
  ports:
    - protocol: TCP
      port: 80
      name: controller
    - protocol: TCP
      port: 8080
      name: admin-web

####################
# traefic  ingress.yaml
####################
[root@hdss7-200 traefik]# cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
```

## 2.3. k8s仪表盘鉴权方式详解

```bash
# openssl 签发证书
[root@hdss7-200 certs]# (umask 077; openssl genrsa -out dashboard.od.com.key 2048)
Generating RSA private key, 2048 bit long modulus
.....................+++
.......................+++
e is 65537 (0x10001)

[root@hdss7-200 certs]# openssl req -new -key dashboard.od.com.key -out dashboard.od.com.csr -subj "/CN=dashboard.od.com/C=CN/ST=BJ/L=Beijing/O=OldboyEdu/OU=ops"

[root@hdss7-200 certs]# ls
apiserver.csr       ca-key.pem            dashboard.od.com.key  kubelet-key.pem
apiserver-csr.json  ca.pem                etcd-peer.csr         kubelet.pem
apiserver-key.pem   client.csr            etcd-peer-csr.json    kube-proxy-client.csr
apiserver.pem       client-csr.json       etcd-peer-key.pem     kube-proxy-client-key.pem
ca-config.json      client-key.pem        etcd-peer.pem         kube-proxy-client.pem
ca.csr              client.pem            kubelet.csr           kube-proxy-csr.json
ca-csr.json         dashboard.od.com.csr  kubelet-csr.json

[root@hdss7-200 certs]# openssl x509 -req -in dashboard.od.com.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out dashboard.od.com.crt -days 3650
Signature ok
subject=/CN=dashboard.od.com/C=CN/ST=BJ/L=Beijing/O=OldboyEdu/OU=ops
Getting CA Private Key

# 使用cfssl查看一下
[root@hdss7-200 certs]# openssl x509 -req -in dashboard.od.com.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out dashboard.od.com.crt -days 3650
Signature ok
subject=/CN=dashboard.od.com/C=CN/ST=BJ/L=Beijing/O=OldboyEdu/OU=ops
Getting CA Private Key
[root@hdss7-200 certs]# 
[root@hdss7-200 certs]# cfssl-certinfo -cert dashboard.od.com.crt 
{
  "subject": {
    "common_name": "dashboard.od.com",
    "country": "CN",
    "organization": "OldboyEdu",
    "organizational_unit": "ops",
    "locality": "Beijing",
    "province": "BJ",
    "names": [
      "dashboard.od.com",
      "CN",
      "BJ",
      "Beijing",
      "OldboyEdu",
      "ops"
    ]
  },
  "issuer": {
    "common_name": "OldboyEdu",
    "country": "CN",
    "organization": "od",
    "organizational_unit": "ops",
    "locality": "beijing",
    "province": "beijing",
    "names": [
      "CN",
      "beijing",
      "beijing",
      "od",
      "ops",
      "OldboyEdu"
    ]
  },
  "serial_number": "17675677124918986791",
  "not_before": "2020-11-20T10:13:14Z",
  "not_after": "2030-11-18T10:13:14Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "",
  "subject_key_id": "",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIDRTCCAi0CCQD1TJ6yA0tIJzANBgkqhkiG9w0BAQsFADBgMQswCQYDVQQGEwJD\nTjEQMA4GA1UECBMHYmVpamluZzEQMA4GA1UEBxMHYmVpamluZzELMAkGA1UEChMC\nb2QxDDAKBgNVBAsTA29wczESMBAGA1UEAxMJT2xkYm95RWR1MB4XDTIwMTEyMDEw\nMTMxNFoXDTMwMTExODEwMTMxNFowaTEZMBcGA1UEAwwQZGFzaGJvYXJkLm9kLmNv\nbTELMAkGA1UEBhMCQ04xCzAJBgNVBAgMAkJKMRAwDgYDVQQHDAdCZWlqaW5nMRIw\nEAYDVQQKDAlPbGRib3lFZHUxDDAKBgNVBAsMA29wczCCASIwDQYJKoZIhvcNAQEB\nBQADggEPADCCAQoCggEBANUSg8x0hgUV35O3oWWlZYE/ErcrBGH/O3Q4CpKBHx3l\nEC5Ku152y59AOsIOtImtoBHON0rKhEr+TzUPco8g+Lq+LV1oG64yZLYzaapHzxsc\nPH+z6bPFTAhhqmeie0mmfdFnBGXpa6OO3p19/D6q0dDAjyYOpaYuMo2qjcjOnC+f\n8Eye3tOzEilWvG1O4mwY9m612l70emwxtUL0XOCZna1u+mzKWOA20M1c4r2TctO0\nDCyktg0aINeWAU3QQQc9PO7ur4V72p2MxR5g704bjqoi/A64QT3yYLkerFmR9Esk\njmkvQkuy+ppCWXsmdBh6OhLeTTkM5/moAKNlhRm17x0CAwEAATANBgkqhkiG9w0B\nAQsFAAOCAQEAMsNkq05G92cS7Y7fx+CvVj/jkNInArNZsxYgcQwUIIKG72lT2fJ7\nE89v7nrMAuYijf3agKB64CeB0Tfg675XT0yWBOM/Zp8CUBcLjvUYCjZXLVIros9W\nY86AoyCtfm7gXedV7NmmK9Yk7OzHGCX76W2iRiM+9rUUq1nk660zpRLMPCkfyWzc\nsWltK7uA9TLnaOFGhBx5UoM9nm3rfBCr8ygTTBq3L5yonI2fnv4rPwr11DNVQikZ\n9btQN6JRwTfUF6RDpLl1caaMujMmhxIx6zdeefzUT7xXWPirEyHxuJH/pazwTveM\nOs70CmvDDbtfT1Jz4HiWK4U4vfDIjzcuFQ==\n-----END CERTIFICATE-----\n"
}

```

### 2.4. 在hdss7-12.host.com上进行相应的配置

```bash
[root@hdss7-11 nginx]# scp -r certs/ root@10.4.7.12:$PWD
[root@hdss7-11 nginx]# scp -r conf.d/ root@10.4.7.12:$PWD	
```

# 3. K8S集群平滑升级技巧

```bash
# 1. 当前k8s集群版本
[root@hdss7-21 conf]# kubectl get node
NAME                STATUS   ROLES         AGE     VERSION
hdss7-21.host.com   Ready    master,node   7d19h   v1.15.2
hdss7-22.host.com   Ready    master,node   7d19h   v1.15.2

# 2. 观察 先升级Pod少的那个节点
[root@hdss7-21 conf]# kubectl get pods -n kube-system -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
coredns-6b6c4f9648-pbrgt                1/1     Running   1          2d16h   172.7.21.3   hdss7-21.host.com   <none>           <none>
kubernetes-dashboard-76dcdb4677-k4jd5   1/1     Running   0          25m     172.7.21.5   hdss7-21.host.com   <none>           <none>
traefik-ingress-hndqx                   1/1     Running   1          42h     172.7.21.4   hdss7-21.host.com   <none>           <none>
traefik-ingress-x5cdk                   1/1     Running   1          42h     172.7.22.2   hdss7-22.host.com   <none>           <none>

# 3. 先升级hdss7-22.host.com
# 3.1. 先注释要升级的节点
[root@hdss7-11 ~]# clear
[root@hdss7-11 ~]# vi /etc/nginx/nginx.conf
...
stream {
    upstream kube-apiserver {
        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
#        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
[root@hdss7-11 ~]# vi /etc/nginx/conf.d/od.com.conf
[root@hdss7-11 ~]# cat /etc/nginx/conf.d/od.com.conf 
upstream default_backend_traefik {
    server 10.4.7.21:81		max_fails=3	fail_timeout=10s;
#    server 10.4.7.22:81		max_fails=3	fail_timeout=10s;
}
server {
	server_name *.od.com;
	
	location / {
		proxy_pass http://default_backend_traefik;
		proxy_set_header Host	$http_host;
		proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
	}
}

# nginx -t
# nginx -s reload

# 3.2. 摘除节点
[root@hdss7-22 ~]# kubectl delete node hdss7-22.host.com
node "hdss7-22.host.com" deleted

# 4. 观察集群万全没有影响，只是22节点不在集群里面了
[root@hdss7-22 ~]# dig -t A kubernetes.default.svc.cluster.local @192.168.0.2 +short
192.168.0.1

# 5. 准备软件包（kubernetes-server-linux-amd64.tar.gz 这个软件包被官网上不让下了，这里模拟用了一个和原来一样的1.15.2的版本进行模拟升级）
[root@hdss7-22 opt]# mkdir new-1.15.2
[root@hdss7-22 opt]# cd src
[root@hdss7-22 src]# tar xvf kubernetes-server-linux-amd64.tar.gz -C /opt/new-1.15.2/
[root@hdss7-22 new-1.15.2]# mv kubernetes/ ../kubernetes-v1.15.4-fade
[root@hdss7-22 bin]# rm -rf *.tar
[root@hdss7-22 bin]# rm -rf *_tag
[root@hdss7-22 opt]# rm -rf kubernetes
[root@hdss7-22 opt]# ln -s kubernetes-v1.15.4-fade/ /opt/kubernetes

# 6. 拷贝证书和相关配置
[root@hdss7-22 bin]# mkdir certs
[root@hdss7-22 bin]# mkdir conf
[root@hdss7-22 bin]# cd certs
[root@hdss7-22 certs]# cp /opt/kubernetes-v1.15.2/server/bin/certs/* .
[root@hdss7-22 certs]# cd ..
[root@hdss7-22 bin]# cd conf
[root@hdss7-22 conf]# cp /opt/kubernetes-v1.15.2/server/bin/conf/* .
# 拷贝相关的sh
[root@hdss7-22 bin]# cp /opt/kubernetes-v1.15.2/server/bin/*.sh .
[root@hdss7-22 bin]# ls
apiextensions-apiserver   hyperkube          kube-controller-manager     kubelet.sh      kube-scheduler.sh
certs                     kubeadm            kube-controller-manager.sh  kube-proxy      mounter
cloud-controller-manager  kube-apiserver     kubectl                     kube-proxy.sh
conf                      kube-apiserver.sh  kubelet                     kube-scheduler

# 重启集群（生产上需要一个一个启动）
supervisorctl restart all

# 查看升级后的状态
[root@hdss7-22 flanneld]# supervisorctl status
etcd-server-7-22                 RUNNING   pid 96269, uptime 1:18:03
flanneld-7-22                    RUNNING   pid 126980, uptime 0:00:38
kube-apiserver-7-22              RUNNING   pid 96264, uptime 1:18:03
kube-controller-manager-7-22     RUNNING   pid 96267, uptime 1:18:03
kube-kubelet-7-22                RUNNING   pid 96262, uptime 1:18:03
kube-proxy-7-22                  RUNNING   pid 96266, uptime 1:18:03
kube-scheduler-7-22              RUNNING   pid 96268, uptime 1:18:03

# 注意，这里hdss7-22.host.com原来是有tag标签的，模拟升级之后已经没有了
[root@hdss7-22 flanneld]# kubectl get node
NAME                STATUS   ROLES         AGE     VERSION
hdss7-21.host.com   Ready    master,node   7d22h   v1.15.2
hdss7-22.host.com   Ready    <none>        78m     v1.15.2

# 取消注释，恢复流量
[root@hdss7-11 ~]# vi /etc/nginx/nginx.conf
[root@hdss7-11 ~]# vi /etc/nginx/conf.d/od.com.conf
```



