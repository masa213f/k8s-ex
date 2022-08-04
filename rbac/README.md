# RBAC

RBACの確認をします。

## 手順概略

1. ServiceAccountを作り、Podを起動する
2. Pod中でkubectlを実行してみる
3. ServiceAccountにRBACを設定する
4. 再び、Pod中でkubectlを実行してみる

### 準備

動作確認で使うNamespaceを作ります。

```console
$ kubectl create namespace ex-rbac
```

### 1. ServiceAccountを作り、Podを起動する

次のマニフェストを適用します。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["sleep", "infinity"]
  serviceAccountName: my-sa
```

ServiceAccountとPodが作られていることを確認します。

```console
$ kubectl apply -n ex-rbac -f <マニフェストのパス>
serviceaccount/my-sa created
pod/test-pod created

# ServiceAccountとPodを確認する
$ kubectl get serviceaccount,pod -n ex-rbac
NAME                     SECRETS   AGE
serviceaccount/default   0         4m2s
serviceaccount/my-sa     0         38s

NAME           READY   STATUS    RESTARTS   AGE
pod/test-pod   1/1     Running   0          38s
```

### 2. Pod中でkubectlを実行してみる

次のように、Podの中で`kubectl`コマンドを実行し、Kubernetesの権限を確認します。

```console
# kubectlをインストールする
$ kubectl exec -it -n ex-rbac test-pod -- bash
root@test-pod:/# apt update -y && apt install -y curl
root@test-pod:/# curl -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl
root@test-pod:/# chmod +x kubectl

# 実行してみる
root@test-pod:/# ./kubectl get pod
Error from server (Forbidden): pods is forbidden: 
  User "system:serviceaccount:ex-rbac:my-sa" cannot list resource "pods" in API group "" in the namespace "ex-rbac"
(権限がないのでGetできない)

# 権限を確認する
root@test-pod:/# ./kubectl auth can-i get pod
no
root@test-pod:/# ./kubectl auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
                                                [/api]                                []               [get]
                                                [/apis/*]                             []               [get]
                                                [/apis]                               []               [get]
                                                [/healthz]                            []               [get]
                                                [/healthz]                            []               [get]
                                                [/livez]                              []               [get]
                                                [/livez]                              []               [get]
                                                [/openapi/*]                          []               [get]
                                                [/openapi]                            []               [get]
                                                [/openid/v1/jwks]                     []               [get]
                                                [/readyz]                             []               [get]
                                                [/readyz]                             []               [get]
                                                [/version/]                           []               [get]
                                                [/version/]                           []               [get]
                                                [/version]                            []               [get]
                                                [/version]                            []               [get]

```

### 3. ServiceAccountにRBACを設定する

次のマニフェストを適用します。
動作確認で使っているServiceAccount(`my-sa`)に、Podの閲覧権限を与えます。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-rolebinding
subjects:
  - kind: ServiceAccount
    name: my-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```
$ kubectl apply -n ex-rbac -f <マニフェストのパス>
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/my-rolebinding created

$ kubectl get role,rolebinding -n ex-rbac
NAME                                        CREATED AT
role.rbac.authorization.k8s.io/pod-reader   2022-08-21T07:48:22Z

NAME                                                   ROLE              AGE
rolebinding.rbac.authorization.k8s.io/my-rolebinding   Role/pod-reader   11s
```

### 4. 再び、Pod中でkubectlを実行してみる

再度、Podの中で`kubectl`を実行してみます。

```console
# 今度はpodリソースが確認できる
$ kubectl exec -it -n ex-rbac test-pod -- bash
root@test-pod:/# ./kubectl get pod
NAME       READY   STATUS    RESTARTS   AGE
test-pod   1/1     Running   0          10m

# 権限も付与されている
root@test-pod:/# ./kubectl auth can-i get pod
yes
root@test-pod:/# ./kubectl auth can-i --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
...
pods                                            []                                    []               [get watch list]
...
```
