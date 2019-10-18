
# 1 角色分类
Kubernetes管理账号分为：serviceaccount(服务账户)和useraccount（用户账户）
## 1.1 useraccount
超级管理员，自定义管理员都属于此类，属于集群外部访问集群，默认应该不需要单独创建用户，而是在创建绑定角色的时候定义
### 1.1.1 用户
### 1.1.2 用户组
## 1.2 serviceaccount
一般用于pod访问集群，比如监控（Prometheus），webui（Dashboard），他们本质是一个pod容器，pod容器访问集群一样是需要认证和授权的，就算普通的pod容器默认都会自带一个serviceaccount,当然这个权限会比较低，而监控和ui需要的权限相对较高
```
Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from default-token-7nb6z (ro)

```

# 2 Role和 ClusterRole
```
apiGroup：支持的API组列表，例如：APIVersion: batch/v1、APIVersion: extensions:v1、apiVersion:apps/v1等，空字符串表示核心API群
resources：支持的资源对象列表，例如：pods、deployments、jobs等
verbs：对资源对象的操作方法列表，例如：get、watch、list、delete、replace、patch等
```
## 2.1 Role
一个名叫dev的角色，作用范围dev的namespace，权限范围是pod，能执行的动作包括：get，watch，list
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: dev
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  ```
## 2.2 ClusterRole
一个名叫secret-reader的集群角色，作用范围是所有namespace，权限范围是secrets，能执行的动作包括：get，watch，list
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  # 集群角色不需要namespace，因为他是作用于所有命名空间
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```
## 2.3 默认集群角色(ClusterRole）
# 3 RoleBinding和ClusterRoleBinding
RoleBinding是将Role中定义的权限授予给用户或用户组。它包含一个subjects列表(users，groups ，service accounts)，并引用该Role。RoleBinding在某个namespace 内授权，ClusterRoleBinding适用在集群范围内使用。
## 3.1 RoleBinding
### 3.1.1 角色绑定用户
一个名叫dev的角色绑定，作为范围是dev命令空间，绑定给了一个用户xk-dev（也可以是其他组或者service accounts），
这个用户具有角色dev的权限
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev
  namespace: dev
subjects:
- kind: User
  name: xk-dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev
  apiGroup: rbac.authorization.k8s.io
```
### 3.1.2 角色绑定组
### 3.1.3 角色绑定service accounts
## 3.2 ClusterRoleBinding
一个名叫read-secrets-global的集群角色绑定，作为范围是全部命名空间，绑定给了一个叫做manager的组，和集群角色secret-reader进行了绑定，具有集群角色secret-reader的权限
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```
# 4 给一个用户配置kubectl权限
## 4.1 创建用户安全证书
### 4.1.1 生成私钥(比如用户：release)
```
openssl genrsa -out release.key 2048
Generating RSA private key, 2048 bit long modulus
.............................+++
...........................................+++
e is 65537 (0x10001)
ll
total 4
-rw-r--r-- 1 root root 1679 Sep  5 11:32 release.key

```
### 4.1.2 创建证书签名请求(csr) CN(这里是用户名) O(这里是组)要显示指明
```
openssl req -new -key release.key -out release.csr -subj "/CN=release/O=k8s"
ll
total 8
-rw-r--r-- 1 root root  907 Sep  5 11:34 release.csr
-rw-r--r-- 1 root root 1679 Sep  5 11:32 release.key

```
### 4.1.3 给用户release签发证书
注意：这里需要经过集群的ca签发，同一个ca签发的证书才会被认（ca签发的集群证书和ca签发的kubectl证书）
```
openssl x509 -req -in release.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out release.crt -days 3650
 ll
total 12
-rw-r--r-- 1 root root  993 Sep  5 11:37 release.crt
-rw-r--r-- 1 root root  907 Sep  5 11:34 release.csr
-rw-r--r-- 1 root root 1679 Sep  5 11:32 release.key

```
## 4.2 生成 KUBECONFIG 文件
### 4.2.1 声明api地址（设置变量）
```
export KUBE_APISERVER=https://192.168.2.240:6443
```
### 4.2.2 设置k8s 集群信息
```
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=release.kubeconfig
```
### 4.2.3 设置用户安全凭证
```
kubectl config set-credentials release \
  --client-certificate=/etc/kubernetes/pki/release.crt \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/release.key \
  --kubeconfig=release.kubeconfig
```

### 4.2.4 设置 context 参数
```
kubectl config set-context release \
  --cluster=kubernetes \
  --user=release \
  --kubeconfig=release.kubeconfig
```

### 4.2.5 设置默认 context
```
kubectl config use-context release \
  --kubeconfig=release.kubeconfig
```

### 4.2.6 放置confgi文件
```
mv release.kubeconfig /roo/.kube/config
```
## 4.3 创建角色
只给了dev命名空间的几个权限
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
## 4.4 角色权限绑定
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-rolebind
  namespace: dev
subjects:
- kind: User
  name: release
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```
## 4.5验证
### 4.5.1 验证具有的dev的pod权限
```
kubectl get pod -n dev
NAME                             READY   STATUS    RESTARTS   AGE
service-goods-6d868566dc-hdkxj   1/1     Running   0          22h
```
### 4.5.2 验证没有其他命名空间权限
用户“release”无法在命名空间“default”中列出api组“”中的资源“pods”
```
 kubectl get pod
Error from server (Forbidden): pods is forbidden: User "release" cannot list resource "pods" in API group "" in the namespace "default"
[root@jenkins ~]# 

```
