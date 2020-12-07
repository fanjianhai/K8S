# 1. nginx

## 代理harbor -- 二进制安装部署 (3.11) 

- 10.4.7.200

##  四层反向代理vip -- 二进制安装部署 （4.3）  

- 10.4.7.11
- 10.4.7.12

## 服务发现 -- coredns

- 10.4.7.200



# 2. 资源yaml

- 搭建好集群之后查看所有资源

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125174815277.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

- 验证集群查看所有资源

```bash
[root@hdss7-21 ~]# vi /root/nginx-ds.yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:

      - name: my-nginx
        image: harbor.od.com/public/nginx:v1.7.9
        ports:
        - containerPort: 80
        
#===================================================
[root@hdss7-21 ~]# kubectl apply -f /root/nginx-ds.yaml 
daemonset.extensions/nginx-ds created

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125175152548.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

- 手动创建pod控制器

```bash
[root@hdss7-21 ~]# kubectl create deployment nginx-dp --image=harbor.od.com/public/nginx:v1.7.9 -n kube-public
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125180726564.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbmppYW5oYWk=,size_16,color_FFFFFF,t_70#pic_center)

