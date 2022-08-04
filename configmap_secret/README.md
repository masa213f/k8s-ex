# ConfigMapとSecretの動作確認

ConfigMapとSecretの動作確認をします。

## 手順概略

1. `kubectl create configmap`でConfigMapを作る
2. マニフェストを適用してSecretを作る
3. ConfigMapとSecretをマウントしてみる
4. ConfigMapを変更し、Pod内にマウントしたファイルの変化を確認する
5. ConfigMapを環境変数として読み込む

## 詳細手順

### 準備

動作確認で使うNamespaceを作ります。

```console
$ kubectl create namespace ex-cm
namespace/ex-cm created
```

### 1. `kubectl create configmap`でConfigMapを作る

次のコマンドを実行してConfigMapを作ります。

```console
$ kubectl create configmap -n ex-cm my-config \
    --from-literal=NEW_NAME=MyakuMyaku \
    --from-literal=OLD_NAME=inochinokagayaki
```

`kubectl create configmap`に`--from-literal=<key>=<value>`を指定すると、オプションで指定したデータを持つConfigMapが生成さます。
生成後のConfigMapのマニフェストを表示すると、以下のように2つのキー(`NEW_NAME`、`OLD_NAME`)が含まれているはずです。

```console
$ kubectl get configmap -n ex-cm my-config -o yaml
...
data:
  NEW_NAME: MyakuMyaku
  OLD_NAME: inochinokagayaki
...
```

### 2. マニフェストを適用してSecretを作る

次のマニフェストをKubernetesクラスタに適用します。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  fuga.txt: ZnVnYT1GVUdBCg==
stringData:
  hoge.txt: |
    hoge=HOGE
    hogehoge=HOGEHOGE
```

```console
$ kubectl apply -n ex-cm -f <マニフェストのパス>
```

生成後のSecretを見ると、次のように`.stringData`がBase64にエンコードされ、`.data`にマージされていることが分かります。

```console
$ kubectl get secret -n ex-cm my-secret -o yaml
...
data:
  fuga.txt: ZnVnYT1GVUdBCg==
  hoge.txt: aG9nZT1IT0dFCmhvZ2Vob2dlPUhPR0VIT0dFCg==
...
```

### 3. ConfigMapとSecretをマウントしてみる

次のマニフェストを適用します。
手順1、2で作ったConfigMapやSecretをマウントしたPodを作ります。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - mountPath: /my-config
      name: my-config-volume
    - mountPath: /my-secret
      name: my-secret-volume
  restartPolicy: Always
  volumes:
  - name: my-config-volume
    configMap:
      name: my-config
  - name: my-secret-volume
    secret:
      secretName: my-secret
```

```console
$ kubectl apply -n ex-cm -f <マニフェストのパス>

# Runningになることを確認する
$ kubectl get pod -n ex-cm
NAME       READY   STATUS    RESTARTS   AGE
test-pod   1/1     Running   0          28s
```

Podに入って、どのようにマウントされるか確認しましょう。

```console
$ kubectl exec -it -n ex-cm test-pod -- bash

# 次のコマンドを実行して、ファイルの内容を確認する
root@test-pod:/# ls /my-config/
NEW_NAME  OLD_NAME
root@test-pod:/# ls /my-secret/
fuga.txt  hoge.txt

# 次のコマンドを実行して、マウントされているファイルシステムを確認する
root@test-pod:/# mount | grep my-
/dev/mapper/ubuntu--vg-ubuntu--lv on /my-config type ext4 (ro,relatime)
tmpfs on /my-secret type tmpfs (ro,relatime,size=10204200k)
```

### 4. ConfigMapを変更し、Pod内にマウントしたファイルの変化を確認する

`kubectl edit`でConfigMapの内容を変更します。

```console
$ kubectl edit configmap -n ex-cm my-config
(省略)
data:
  OLD_NAME: inochinokagayaki   # .data以下の値を変更してみる
  NEW_NAME: MyakuMyaku         #
(省略)
```

Podの中のファイルが変更されるので、確認してみましょう。

> **Note**
>
> 自分が確認した時は、ConfigMap変更してからコンテナのファイルが更新されるまで、1分程度かかりました。

### 5. ConfigMapを環境変数として読み込む

次のマニフェストを適用する。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod2
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["sleep", "infinity"]
    envFrom:
    - configMapRef:
        name: my-config
```

```console
$ kubectl apply -n ex-cm -f <マニフェストのパス>

# test-pod2がRunningになることを確認する
$ kubectl get pod -n ex-cm
NAME        READY   STATUS    RESTARTS   AGE
test-pod    1/1     Running   0          7m54s
test-pod2   1/1     Running   0          10s   # こちら
```

ConfigMapに含まれるデータが環境変数になっていることを確認する。

```console
$ kubectl exec -it -n ex-cm test-pod2 -- printenv
...
NEW_NAME=MyakuMyaku
OLD_NAME=inochinokagayaki
...
```
