# emptyDirの動作確認

emptyDirの動作を確認します。

## 手順概略

1. 1つのemptyDirを、2つのコンテナでマウントしているPodを作る
2. 一方のコンテナからemptyDirの中にファイルを作り、もう一方のコンテナからファイルを探してみる
3. ファイルを作ったコンテナを再起動させる
4. 作成したファイルがあるか確認する

## 詳細手順

### 準備

動作確認で使うNamespaceを作ります。

```console
$ kubectl create namespace ex-emptydir
namespace/ex-emptydir created
```

### 1. 1つのemptyDirを、2つのコンテナでマウントしているPodを作る

次のマニフェストを適用します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: my-jammy
    image: ubuntu:jammy
    command: ["bash", "-c", "trap 'kill $(jobs -p)' EXIT; sleep 9999"]
    volumeMounts:
    - mountPath: /mount-point-in-jammy
      name: my-volume

  - name: my-focal
    image: ubuntu:focal
    command: ["bash", "-c", "trap 'kill $(jobs -p)' EXIT; sleep 8888"]
    volumeMounts:
    - mountPath: /mount-point-in-focal
      name: my-volume

  restartPolicy: Always
  volumes:
  - name: my-volume
    emptyDir: {}
```

適用後、PodがRunningになることを確認します。

```console
$ kubectl apply -n ex-emptydir -f <マニフェストのパス>
pod/test-pod created

$ kubectl get pod -n ex-emptydir
NAME       READY   STATUS    RESTARTS   AGE
test-pod   2/2     Running   0          22s
```

### 2. 一方のコンテナからemptyDirの中にファイルを作り、もう一方のコンテナからファイルを探してみる

1つめのコンテナ`my-jammy`に入ってemptyDirの中にファイルを作る。

```console
$ kubectl exec -it -n ex-emptydir test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal

# Ubuntu 22.04であることを確認しておく
root@test-pod:/# grep VERSION= /etc/os-release
VERSION="22.04.1 LTS (Jammy Jellyfish)"

# ファイルを作ってみる
root@test-pod:/# echo Hello > /mount-point-in-jammy/from-jammy
root@test-pod:/# cat /mount-point-in-jammy/from-jammy
Hello
```

次に、2つめのコンテナ`my-focal`に入って、ファイルを確認してみる。


```console
# コンテナ名を指定(-c my-focal)して、my-focalコンテナに入る
$ kubectl -n ex-emptydir exec -it test-pod -c my-focal -- bash

# Ubuntuのバージョンを確認してみる
root@test-pod:/# grep VERSION= /etc/os-release
VERSION="20.04.4 LTS (Focal Fossa)"

# my-jammyで作ったファイルを確認する
root@test-pod:/# ls /mount-point-in-focal/
from-jammy
root@test-pod:/# cat /mount-point-in-focal/from-jammy
Hello
```

### 3. ファイルを作ったコンテナを再起動させる

`my-jammy`のPID 1のプロセスを終了し、コンテナを終了させます。

```console
$ kubectl exec -it -n ex-emptydir test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal

root@test-pod:/# kill 1
root@test-pod:/# command terminated with exit code 137
```

restartしていることを確認します。

```
$ kubectl get pod -n ex-emptydir
NAME       READY   STATUS    RESTARTS      AGE
test-pod   2/2     Running   1 (20s ago)   7m31s
```

### 4. 作成したファイルがあるか確認する

```console
$ kubectl exec -it -n ex-emptydir test-pod -- bash
Defaulted container "my-jammy" out of: my-jammy, my-focal

# ファイルを確認する
root@test-pod:/# ls /mount-point-in-jammy/
from-jammy
root@test-pod:/# cat /mount-point-in-jammy/from-jammy 
Hello
```
