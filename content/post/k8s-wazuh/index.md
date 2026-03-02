---
title: KubernetesにWazuh構築してみた
date: 2025-11-29
tags: 
    - Kubernetes
    - Wazuh
categories:
    - 自宅サーバー
---

サーバーなどに潜む脅威を検出・分析してくれるWazuhというソフトを知った。  
Kustomizeも提供されているのでKubernetes上に構築しようと思う。  
[https://wazuh.com/](https://wazuh.com/)  
[https://github.com/wazuh/wazuh-kubernetes](https://github.com/wazuh/wazuh-kubernetes)

### Kustomizeでインストール

まず、[GitHub](https://github.com/wazuh/wazuh-kubernetes/tree/main/wazuh)からデプロイ用のファイルをダウンロードしてきます。  
wazuhフォルダだけとりあえず使うためArgoCDに登録しているリポジトリ内に配置します。

次にSSLの証明書を生成します。

```bash
bash certs/dashboard_http/generate_certs.sh
bash certs/indexer_cluster/generate_certs.sh
```

続けてマニフェストを編集していきます。

```yaml {title="kustomization.yml"}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Adds wazuh namespace to all resources.
namespace: wazuh

# ~~省略~~
resources:
  - base/wazuh-ns.yaml
#既存のものを使うため削除
  # - base/storage-class.yaml
# ~~省略~~
```

既存のStorageClassを使うように置き換えます。

```yaml {title="indexer-sts.yaml"}
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: wazuh-indexer
  namespace: wazuh
# ~~省略~~
  volumeClaimTemplates:
    - metadata:
        name: wazuh-indexer
        namespace: indexer-cluster
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: <既存のものに置き換える>
        resources:
          requests:
            storage: 500Mi
```

```yaml {title="wazuh-master-sts.yaml"}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-manager-master
  namespace: wazuh
# ~~省略~~
  volumeClaimTemplates:
    - metadata:
        name: wazuh-manager-master
        namespace: wazuh
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: <既存のものに置き換える>
        resources:
          requests:
            storage: 500Mi
```

```yaml {title="wazuh-worker-sts.yaml"}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-manager-worker
  namespace: wazuh
# ~~省略~~
  volumeClaimTemplates:
    - metadata:
        name: wazuh-manager-worker
        namespace: wazuh
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: <既存のものに置き換える>
        resources:
          requests:
            storage: 500Mi
```

[local-env](https://github.com/wazuh/wazuh-kubernetes/tree/main/envs/local-env) を見るとパッチがあるので直接変更しておきます。

```yaml {title="indexer-sts.yaml"}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-indexer
  namespace: wazuh
spec:
  replicas: 1 #3→1
# ~~省略~~
      containers:
        - name: wazuh-indexer
          image: 'wazuh/wazuh-indexer:5.0.0'
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1
              memory: 2Gi
# ~~省略~~
```

```yaml {title="wazuh-worker-sts.yaml"}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wazuh-manager-worker
  namespace: wazuh
spec:
  replicas: 1 #2→1
# ~~省略~~
```

ここで一度デプロイします。

### 設定の修正

ダッシュボード等にアクセスできるようにするために設定を修正します。

podに入りhashを生成します。

```bash
kubectl exec -it wazuh-indexer-0 -n wazuh -- /bin/bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

取得したhashで上書きします。

```yaml {title="internal_users.yml"}
# ~~省略~~
admin:
  hash: "<取得したhash>"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "<取得したhash>"
  reserved: true
  description: "Demo kibanaserver user"
# ~~省略~~
```

```yaml {title="indexer-cred-secret.yaml"}
apiVersion: v1
kind: Secret
metadata:
  name: indexer-cred
data:
  username: YWRtaW4=              # string "admin" base64 encoded
  password: U2VjcmV0UGFzc3dvcmQ=  # パスワードをbase64にしたもの
```

```yaml {title="dashboard-cred-secret.yaml"}
apiVersion: v1
kind: Secret
metadata:
    name: dashboard-cred
data:
    username: a2liYW5hc2VydmVy  # string "kibanaserver" base64 encoded
    password: a2liYW5hc2VydmVy  # パスワードをbase64にしたもの
```

ここで再度デプロイしてください。  
podに入り作業します。

```bash
kubectl exec -it wazuh-indexer-0 -n wazuh -- /bin/bash

export INSTALLATION_DIR=/usr/share/wazuh-indexer/config
CACERT=$INSTALLATION_DIR/certs/root-ca.pem
KEY=$INSTALLATION_DIR/certs/admin-key.pem
CERT=$INSTALLATION_DIR/certs/admin.pem
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd /usr/share/wazuh-indexer/config/opensearch-security/ -nhnv -cacert  $CACERT -cert $CERT -key $KEY -p 9200 -icl -h $NODE_NAME
```

最後にpodをすべて削除し反映させれば完了です。

### ダッシュボードにアクセス

```
https://<割り当てられたip>
```

でアクセスできます。

![Wazuhダッシュボード](https://mtayo.net/wp-content/uploads/2025/11/image-1024x547.png)

Usernameはadmin、Passwordは先ほど設定したものです。

### 参考

[K8sにWazuhをインストールしてセキュアな環境を作る](https://qiita.com/yuito_it_/items/4852ef7f6d239244ee90)