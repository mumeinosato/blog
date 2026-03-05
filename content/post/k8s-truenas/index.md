---
title: TrueNASをKubernetesのストレージとして使う
date: 2025-11-29
tags: 
    - Kubernetes
    - TrueNAS
categories:
    - 自宅サーバー
---

## 概要
TrueNASの仕様変更によりdemocratic-csiのデフォルトでは利用できなくなっていたので修正方法を記述しておく。

## TrueNASの設定

[こちら](https://zenn.dev/imksoo/articles/f52a824c5ea632)の記事を参考にdemocratic-csiのリポジトリの追加とTrueNASの設定を行ってください。

## democratic-csiの設定

ZFSコマンドのパスを指定することで動作させることができます。

```yaml
# truenas-iscsi.yml
csiDriver:
  name: "org.democratic-csi.iscsi"
storageClasses:
  - name: truenas-iscsi-csi
    defaultClass: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    allowVolumeExpansion: true
    parameters:
      fsType: ext4
    mountOptions: []
    secrets:
      provisioner-secret:
      controller-publish-secret:
      node-stage-secret:
      node-publish-secret:
      controller-expand-secret:
driver:
  config:
    driver: freenas-iscsi
    httpConnection:
      protocol: http
      host: 172.16.0.100 #TrueNASのIP
      port: 80
      username: root
      password: PASSWORD
    sshConnection:
      host: 172.16.0.100
      port: 22
      username: root
      password: PASSWORD
    zfs:
      cli:
        paths:
          zfs: /usr/sbin/zfs
          zpool: /usr/sbin/zpool
      datasetParentName: main/k8s-pvs
      detachedSnapshotsDatasetParentName: main/k8s-snaps
    iscsi:
      targetPortal: 172.16.0.100:3260
      namePrefix: csi-
      targetGroups:
        - targetGroupPortalGroup: 1
          targetGroupInitiatorGroup: 1
          targetGroupAuthType: None
          targetGroupAuthGroup:
      extentInsecureTpc: true
      extentXenCompat: false
      extentDisablePhysicalBlocksize: true
      extentBlocksize: 4096
      extentRpm: "5400"
      extentAvailThreshold: 0
```

以下のコマンドでdemocratic-csi をデプロイしてください。

```bash
helm upgrade --install --values truenas-iscsi.yml --create-namespace --namespace democratic-csi trunas-iscsi democratic-csi/democratic-csi
```

## 参考

[Kubernetes に democratic-csi を入れて、TrueNAS に自動的にPVを作ってもらうようにした](https://zenn.dev/imksoo/articles/f52a824c5ea632)