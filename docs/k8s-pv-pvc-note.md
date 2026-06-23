# Kubernetes / OpenShift PV、PVC、NFS、StatefulSet 

## 結論

`PV/PVC` 可以先用一句話記住：

> **Pod 要存資料 → 找 PVC 申請 → PVC 綁 PV → PV 背後接 NFS / Ceph / vSphere / 磁碟 → Pod 重建後資料還在。**

最簡單關係：

```text
Pod → PVC → PV → 後端儲存
```

口語化比喻：

| Kubernetes 物件 | 比喻 |
|---|---|
| `Pod` | 要存資料的程式 / 租客 |
| `PVC` | 儲存申請單 / 租屋申請單 |
| `PV` | 實際分配到的儲存空間 / 房子 |
| `NFS` | 真正放資料的位置 / 房子實際所在地 |
| `StorageClass` | 儲存方案 / 房型方案 |
| `StatefulSet` | 幫每個 Pod 固定配一份資料與名字的管理員 |

---

## 1. Pod 為什麼需要持久化儲存

`Pod` 是臨時的，可能會被刪除、重建、搬到別的 Node。

如果資料只放在 Pod 裡面：

```text
Pod 被刪掉 → 資料可能一起消失
```

所以 Kubernetes 要把「程式」和「資料」分開：

```text
Pod = 跑程式
PV/PVC = 放資料
```

口語化：

```text
Pod 像臨時工
資料像工作成果

臨時工可以換人
但工作成果不能不見

所以資料不能只放在 Pod 裡
要放到外部儲存
```

常見需要持久化儲存的服務：

| 類型 | 為什麼需要 PVC |
|---|---|
| 資料庫 | MySQL、PostgreSQL、MongoDB 資料不能跟 Pod 一起消失 |
| 檔案上傳服務 | 使用者上傳的附件、圖片要保留 |
| StatefulSet | 每個 Pod 要有固定資料 |
| NFS 共享資料 | 多個服務需要共用資料 |

---

## 2. PV 是什麼

`PV` 全名是 `PersistentVolume`。

口語化：

```text
PV = Kubernetes / OpenShift 看得到的一塊真正儲存空間
```

但 PV 背後不一定是一顆實體硬碟，也可能是：

```text
NFS
Ceph
vSphere Disk
iSCSI
Cloud Disk
Local Disk
```

在這份筆記的範例中，PV 背後是 NFS：

```text
NFS Server 上有 /nfsdata/1
Kubernetes 把 /nfsdata/1 包裝成一個 PV
Pod 不直接知道 /nfsdata/1
Pod 只透過 PVC 使用它
```

範例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfsdata/1
    server: 10.245.68.72
```

欄位說明：

| 欄位 | 意思 |
|---|---|
| `kind: PersistentVolume` | 這是一個 PV |
| `metadata.name` | PV 名稱 |
| `capacity.storage` | 這塊儲存多大 |
| `accessModes` | 支援哪種讀寫模式 |
| `persistentVolumeReclaimPolicy` | PVC 刪掉後，PV 要怎麼處理 |
| `storageClassName` | 這塊 PV 屬於哪個儲存類別 |
| `nfs.server` | NFS Server IP |
| `nfs.path` | NFS 實際目錄 |

重點：

```text
PV = 管理員或系統準備好的儲存資源
```

---

## 3. PVC 是什麼

`PVC` 全名是 `PersistentVolumeClaim`。

口語化：

```text
PVC = 儲存申請單
```

PVC 不是真正的硬碟。  
PVC 只是跟 Kubernetes 說：

```text
我要 1Gi
我要 ReadWriteOnce
我要 storageClassName=nfs
```

範例：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 1Gi
```

欄位說明：

| 欄位 | 意思 |
|---|---|
| `kind: PersistentVolumeClaim` | 這是一張 PVC 申請單 |
| `metadata.name` | PVC 名稱 |
| `accessModes` | 申請哪種讀寫模式 |
| `storageClassName` | 申請哪種儲存類別 |
| `resources.requests.storage` | 申請多少容量 |

一句話：

```text
PVC = 我要什麼
PV = 實際給你什麼
```

---

## 4. Pod、PVC、PV、NFS 的關係

完整資料流：

```text
Pod
  │
  │ 我要把資料放到 /data
  ▼
PVC
  │
  │ 我要 1Gi + RWO + storageClassName=nfs
  ▼
PV
  │
  │ 找到符合條件的 nfspv1
  ▼
NFS
  │
  │ 實際資料寫到 10.245.68.72:/nfsdata/1
  ▼
資料真的被保存
```

口語化：

```text
Pod：我要硬碟
PVC：我幫你申請
PV：我這裡有一塊符合的
NFS：資料最後放我這裡
```

最重要：

```text
Pod 不直接掛 PV
Pod 掛 PVC
PVC 再綁 PV
PV 背後接 NFS / Ceph / vSphere / 磁碟
```

---

## 5. PVC 怎麼找到 PV

PVC 要綁定 PV，主要看三個條件：

```text
capacity
accessModes
storageClassName
```

### 5.1 capacity：容量

PVC 要 `1Gi`，PV 至少要 `1Gi`。

| PVC 要求 | PV 容量 | 結果 |
|---|---:|---|
| `1Gi` | `0.9Gi` | 不行，太小 |
| `1Gi` | `1Gi` | 可以 |
| `1Gi` | `1.2Gi` | 可以，但多給一點 |

### 5.2 accessModes：讀寫模式

PVC 要的讀寫模式，PV 要能支援。

常見模式：

```text
ReadWriteOnce
ReadOnlyMany
ReadWriteMany
```

### 5.3 storageClassName：儲存類別

名稱要一致。

| PVC | PV | 結果 |
|---|---|---|
| `nfs` | `nfs` | 可以 |
| `nfs` | `nfs1` | 不行 |
| `standard` | `nfs` | 不行 |

範例：

```text
PVC 要求：
1Gi
ReadWriteOnce
storageClassName=nfs
```

| PV | 容量 | 模式 | 類別 | 可不可綁 |
|---|---:|---|---|---|
| `nfspv1` | `1Gi` | `RWO` | `nfs` | 可以 |
| `nfspv2` | `0.9Gi` | `RWO` | `nfs` | 不行，容量太小 |
| `nfspv3` | `1.2Gi` | `RWO` | `nfs` | 可以 |
| `nfspv5` | `1Gi` | `RWX` | `nfs` | 視 PVC 需求與後端支援而定 |
| `nfspv6` | `1Gi` | `RWO` | `nfs1` | 不行，類別不同 |

---

## 6. RWO、ROX、RWX 怎麼記

原筆記中有一個地方要修正：

> `RWX` 不是多節點只讀。  
> `RWX` 是多節點讀寫。  
> `ROX` 才是多節點只讀。

正確表格：

| 縮寫 | 全名 | 口語化 | 意思 |
|---|---|---|---|
| `RWO` | `ReadWriteOnce` | 一邊寫 | 通常單一 Node 可讀寫 |
| `ROX` | `ReadOnlyMany` | 多邊讀 | 多個 Node 可唯讀 |
| `RWX` | `ReadWriteMany` | 多邊讀寫 | 多個 Node 可讀寫 |
| `RWOP` | `ReadWriteOncePod` | 單一 Pod 專用 | 只允許一個 Pod 讀寫 |

最短記法：

```text
RWO = 一個地方寫
ROX = 很多地方讀
RWX = 很多地方讀寫
```

常見搭配：

| 儲存類型 | 常見模式 |
|---|---|
| block storage / iSCSI / vSphere Disk | 多半是 `RWO` |
| NFS | 常見 `RWX` |
| CephFS | 常見 `RWX` |
| Ceph RBD | 常見 `RWO` |

---

## 7. 回收策略：Retain、Delete、Recycle

這個是在講：

```text
PVC 刪掉後，PV 和資料要怎麼辦？
```

| 策略 | 口語化 | 結果 |
|---|---|---|
| `Retain` | 資料保留 | PVC 刪掉後，PV 和資料還在，要人工處理 |
| `Delete` | 一起刪掉 | PVC 刪掉後，後端 volume 可能一起刪 |
| `Recycle` | 清空再重用 | 舊做法，已不建議使用 |

現在實務重點：

```text
Retain = 安全，資料保留，但要手動清
Delete = 方便，但刪 PVC 可能連資料一起刪
Recycle = 舊東西，不建議新環境使用
```

學習時先背：

```text
正式環境不要亂刪 PVC
因為 Delete policy 可能真的把後端資料刪掉
```

---

## 8. PV/PVC 狀態怎麼看

### PV 狀態

| 狀態 | 口語化 | 意思 |
|---|---|---|
| `Available` | 空房 | PV 還沒被 PVC 使用 |
| `Bound` | 已出租 | PV 已經綁到 PVC |
| `Released` | 租客走了但還沒整理 | PVC 被刪，但 PV 還沒回到可用狀態 |
| `Failed` | 整理失敗 | 回收失敗 |

### PVC 狀態

| 狀態 | 口語化 | 意思 |
|---|---|---|
| `Pending` | 還沒申請到 | 找不到符合的 PV |
| `Bound` | 申請成功 | 已經綁到 PV |
| `Lost` | 找不到原本 PV | 綁定關係出問題 |

最容易卡的是：

```text
PVC Pending
```

通常代表：

```text
沒有符合條件的 PV
StorageClass 不對
容量不夠
accessModes 不對
CSI / provisioner 有問題
```

---

## 9. StatefulSet 為什麼會自動建立 PVC

`StatefulSet` 是給有狀態應用用的。

例如：

```text
資料庫
主從服務
需要固定資料的服務
需要固定 Pod 名稱的服務
```

`Deployment` 的 Pod 名稱通常是亂數：

```text
nginx-7f8c9d6b4-abcde
```

刪掉重建後名字可能變。

`StatefulSet` 的 Pod 名稱固定：

```text
web-0
web-1
web-2
```

而且它可以透過 `volumeClaimTemplates` 幫每個 Pod 建自己的 PVC。

範例：

```yaml
volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: nfs
      resources:
        requests:
          storage: 1Gi
```

如果 StatefulSet 叫 `web`，Pod 是：

```text
web-0
web-1
web-2
```

那 PVC 會像：

```text
www-web-0
www-web-1
www-web-2
```

意思是：

```text
web-0 有自己的 PVC
web-1 有自己的 PVC
web-2 有自己的 PVC
```

---

## 10. Pod 刪除後資料為什麼還在

因為資料不是存在 Pod 裡。

資料實際流程：

```text
web-0
  │
  │ 寫入 /usr/local/nginx/html/index.html
  ▼
PVC：www-web-0
  ▼
PV：nfspv7
  ▼
NFS：/nfsdata/7/index.html
```

刪掉 `web-0`：

```text
Pod 消失
PVC 還在
PV 還在
NFS 資料還在
```

等 `web-0` 重建：

```text
新的 web-0
重新掛回 www-web-0
所以還看得到原本資料
```

這就是 StatefulSet 的重點：

```text
Pod 可以重建
資料要延續
身份要固定
```

---

## 11. 為什麼 StatefulSet 擴到 10 個會 Pending

範例：

```bash
kubectl scale statefulSet web --replicas=10
```

不是所有 Pod 都一定會成功。  
原因是每個 Pod 都需要一個 PVC，而每個 PVC 都要找到符合條件的 PV。

假設 PVC 要：

```text
1Gi
ReadWriteOnce
storageClassName=nfs
```

但剩下的 PV 是：

| PV | 問題 |
|---|---|
| `nfspv2` | 只有 `0.9Gi`，容量太小 |
| `nfspv5` | 是 `RWX`，不一定符合 PVC 要求 |
| `nfspv6` | `storageClassName=nfs1`，類別不同 |

所以後面的 Pod 會卡住：

```text
Pod 等 PVC
PVC 等 PV
PV 沒有符合的
所以 Pod Pending / ContainerCreating
```

文字圖：

```text
StatefulSet replicas=10
  │
  ├─ web-0 → PVC → PV 成功
  ├─ web-1 → PVC → PV 成功
  ├─ web-2 → PVC → PV 成功
  ├─ web-3 → PVC → PV 成功
  └─ web-4 → PVC 找不到合適 PV → Pending
```

---

## 12. 最小 YAML 範例

### 12.1 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 1Gi
```

### 12.2 Pod 掛 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: nginx:latest
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: demo-pvc
```

欄位重點：

| 欄位 | 意思 |
|---|---|
| `claimName: demo-pvc` | Pod 要掛哪個 PVC |
| `mountPath` | 掛到 container 裡哪個路徑 |
| `volumeMounts.name` | 要跟 `volumes.name` 對上 |
| `storageClassName` | 要跟 PV 或 StorageClass 對上 |
| `resources.requests.storage` | 要多少容量 |

OpenShift / 離線環境注意：

```text
image: nginx:latest
```

正式或離線環境不能假設可拉 Docker Hub，要改成內部 registry image。

---

## 13. 唯讀檢查命令

### 看 PV

```bash
kubectl get pv
```

目的：看叢集有哪些 PV、容量、狀態、StorageClass。  
預期：有 `Available` 或已綁定的 `Bound`。  
失敗代表：沒有可用 PV，PVC 可能會 `Pending`。

OpenShift：

```bash
oc get pv
```

---

### 看 PVC

```bash
kubectl get pvc
```

目的：看 PVC 是否綁到 PV。  
預期：`STATUS` 是 `Bound`。  
失敗代表：`Pending` 表示還沒找到合適 PV。

OpenShift：

```bash
oc get pvc
```

---

### 看 PVC 詳細原因

```bash
kubectl describe pvc <pvc-name>
```

目的：看 PVC Events。  
預期：沒有 binding / provisioning 錯誤。  
失敗代表：通常是容量、accessModes、StorageClass、CSI provisioner 問題。

OpenShift：

```bash
oc describe pvc <pvc-name>
```

---

### 看 Pod 狀態

```bash
kubectl get pod -o wide
```

目的：看 Pod 是否 Running，以及在哪個 Node。  
預期：`Running`。  
失敗代表：如果卡 `ContainerCreating`，可能是 volume 掛載問題。

OpenShift：

```bash
oc get pod -o wide
```

---

### 看 Pod 掛載事件

```bash
kubectl describe pod <pod-name>
```

目的：看 Pod Events。  
預期：沒有 `FailedMount`、`MountVolume`、`AttachVolume`。  
失敗代表：PVC 沒 Bound、NFS 連不到、CSI 掛載失敗、權限問題。

OpenShift：

```bash
oc describe pod <pod-name>
```

---

## 14. 三種常見錯誤怎麼判斷

### A. PVC Pending

口語化：

```text
申請單送出去了，但沒找到硬碟
```

檢查：

```bash
kubectl describe pvc <pvc-name>
kubectl get pv
```

常見原因：

| 原因 | 判斷 |
|---|---|
| 容量不足 | PVC 要 `1Gi`，PV 只有 `0.9Gi` |
| StorageClass 不同 | PVC `nfs`，PV `nfs1` |
| accessModes 不合 | PVC 要的模式，PV 不支援 |
| 沒有 PV | 沒有 `Available` PV |
| 動態供應失敗 | StorageClass / CSI Driver 有問題 |

---

### B. Pod ContainerCreating

口語化：

```text
Pod 要起來了，但硬碟還沒掛好
```

檢查：

```bash
kubectl describe pod <pod-name>
kubectl get pvc
```

常見原因：

| 原因 | 判斷 |
|---|---|
| PVC 還沒 Bound | 先看 `kubectl get pvc` |
| NFS 掛不上 | Events 會有 `FailedMount` |
| Node 無法連 NFS | 特定 Node 上一直失敗 |
| CSI driver 問題 | Events 有 attach / mount 失敗 |
| 權限問題 | 掛上後寫入失敗 |

---

### C. Permission denied

口語化：

```text
硬碟拿到了，但鑰匙不對
```

檢查：

```bash
kubectl exec -it <pod-name> -- id
kubectl exec -it <pod-name> -- ls -ld <mount-path>
kubectl exec -it <pod-name> -- sh -c 'touch <mount-path>/test-write'
```

目的：

| 命令 | 看什麼 |
|---|---|
| `id` | container 內實際 UID/GID |
| `ls -ld` | 目錄 owner/group/permission |
| `touch` | 是否真的能寫入 |

OpenShift 特別注意：

```text
OpenShift 常用隨機 UID
所以你不能假設 container 是 root
也不能假設 chmod 777 / no_root_squash 是正式可接受做法
```

---

## 15. 原筆記中要修正或注意的地方

| 原筆記內容 / 做法 | 判斷 |
|---|---|
| `RWX 多節點只讀` | 錯，`RWX` 是多節點讀寫；`ROX` 才是多節點只讀 |
| `Recycle` | 已過時，不建議新環境使用 |
| `chmod 777` | 實驗方便，正式環境不建議 |
| `no_root_squash` | 實驗方便，正式環境有安全風險 |
| `--force --grace-period=0` 刪 PV | 高風險，不建議正式環境 |
| 手動 edit PV 清除 claimRef | 實驗可理解流程，正式環境要先確認資料與回收策略 |
| 用 NFS 跑資料庫類 StatefulSet | 可學概念，但正式要評估一致性、效能、鎖定與 HA |

---

## 16. 最後總整理

```text
Pod 是跑程式的
PVC 是申請單
PV 是 Kubernetes 看到的儲存資源
NFS / Ceph / vSphere 才是真正放資料的地方

Pod 掛 PVC
PVC 綁 PV
PV 指到後端儲存

PVC Pending = 找不到合適 PV
Pod ContainerCreating = volume 還沒掛好
Permission denied = 掛好了但權限不對

StatefulSet 會幫每個 Pod 建自己的 PVC
Pod 刪掉資料還在
PVC 刪掉後 PV 可能變 Released
```

---

## 17. 五句話記住

```text
1. PVC 不是硬碟，是申請單。
2. PV 才是 Kubernetes 看到的儲存資源。
3. Pod 掛 PVC，不是直接掛 PV。
4. 資料真正存在 NFS / Ceph / vSphere / 磁碟。
5. StatefulSet 會讓每個 Pod 固定掛回自己的 PVC，所以 Pod 重建後資料還在。
```
