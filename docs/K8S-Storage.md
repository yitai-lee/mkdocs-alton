# K8S / OCP Storage Master 筆記

> 目的：給之後會忘記的自己複習。  
> 風格：先建立腦中模型，再看 YAML。  

---

# 0. 先講結論

Kubernetes Storage 整套觀念，其實都在回答一個問題：

```text
Pod 消失之後，資料還在不在？
```

不同 Storage 類型答案不同。

| 類型 | 資料放哪裡 | Pod 刪除後資料 | 主要用途 |
|---|---|---|---|
| emptyDir | Pod 所在 Node 的 kubelet 暫存目錄 | 消失 | 暫存、Cache、同 Pod 內 Container 共用資料 |
| emptyDir memory | Node RAM / tmpfs | 消失 | 高速暫存、小量資料 |
| hostPath | Pod 所在 Node 的指定路徑 | 留在該 Node | 存取 Node 檔案、測試、特殊系統用途 |
| PV / PVC | 外部儲存或叢集儲存 | 通常保留，依 reclaimPolicy | 正式持久化資料 |
| StorageClass | 不直接存資料，是動態建立 PV 的規則 | 看建立出的 PV 設定 | 自動建立 PV |
| NFS PV | NFS Server | 留在 NFS Server | 多 Pod / 多 Node 共用檔案 |

最重要的一句話：

```text
Container 重啟，不等於 Pod 重建。

Container 重啟：Pod 還在，所以 Pod 生命週期內的 Volume 也還在。
Pod 被刪除：Pod 生命週期型 Volume 會消失，例如 emptyDir。
```

---

# 1. 為什麼 Kubernetes 需要 Volume

## 1.1 沒有 Volume 時發生什麼事

Container 內部也有檔案系統。

例如 nginx 在容器內寫檔：

```text
Container
└─ /usr/share/nginx/html/index.html
```

或應用程式寫 log：

```text
Container
└─ /app/logs/app.log
```

問題是：

```text
Container 被刪除 / 重建
↓
新的 Container 從 Image 重新建立
↓
舊 Container 裡寫出來的檔案不一定還在
```

所以如果你把重要資料寫在 Container 自己的檔案層裡，風險很高。

---

## 1.2 Volume 解決什麼問題

Volume 的核心目的不是「一定要永久保存資料」。

Volume 的核心目的是：

```text
把資料從 Container 本身抽離出來。
```

有了 Volume 後，模型變成：

```text
Pod
├─ Container
└─ Volume
```

Container 不再只依賴自己的暫存檔案層，而是把資料寫到 Volume。

---

## 1.3 腦中模型

```text
Pod
├─ Container A
├─ Container B
└─ Volume
```

Volume 屬於 Pod。

Container 只是透過 `volumeMounts` 把 Volume 掛進自己檔案系統裡使用。

一句話：

```text
Volume 是 Pod 的資料區。
Container 是使用者。
volumeMounts 是掛載點。
```

---

## 1.4 YAML 只看最小模型

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  volumes:
    - name: data-volume
      emptyDir: {}

  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data-volume
          mountPath: /data
```

只看三個點：

| 欄位 | 意義 |
|---|---|
| `volumes` | Pod 宣告有哪些 Volume |
| `name: data-volume` | 給這個 Volume 一個名字 |
| `volumeMounts` | Container 把這個 Volume 掛到哪個路徑 |

不要先背完整 YAML。先記：

```text
Pod 先定義 Volume。
Container 再掛載 Volume。
```

---

# 2. Volume、Pod、Container 的生命週期

## 2.1 Container 重啟

假設：

```text
Pod
├─ Container A
└─ Volume
```

Container A crash：

```text
Container A 死掉
↓
Kubelet 重新拉起 Container A
↓
Pod 還是同一個 Pod
↓
Volume 還在
```

所以：

```text
Container 重啟，不代表 Volume 一定消失。
```

---

## 2.2 Pod 被刪除

假設：

```bash
kubectl delete pod demo
```

這不是 Container 重啟。

這是整個 Pod 被刪除：

```text
Pod 消失
↓
Pod 生命週期內的 Volume 跟著消失
```

emptyDir 就是這種。

---

## 2.3 Deployment 自動補 Pod 時

你刪 Deployment 底下的 Pod：

```bash
kubectl delete pod nginx-xxxx
```

Deployment / ReplicaSet 會再補一個新的 Pod。

但注意：

```text
舊 Pod
≠
新 Pod
```

所以如果舊 Pod 用的是 emptyDir：

```text
舊 Pod 被刪
↓
舊 emptyDir 消失
↓
新 Pod 建立
↓
新的 emptyDir
↓
舊資料不會回來
```

這就是 Storage 的核心觀念。

---

# 3. emptyDir

## 3.1 emptyDir 是什麼

emptyDir 是 Kubernetes 最簡單的 Volume。

Pod 被建立時，Kubelet 會幫這個 Pod 建立一個空目錄。

所以叫：

```text
emptyDir = 一開始是空的目錄
```

YAML 最小寫法：

```yaml
volumes:
  - name: data
    emptyDir: {}
```

---

## 3.2 emptyDir 想解決什麼問題

emptyDir 主要解決兩種問題。

第一，同一個 Pod 裡多個 Container 要共用資料。

```text
Pod
├─ App Container
├─ Sidecar Container
└─ emptyDir
```

第二，Container 需要暫存空間。

```text
下載檔案
↓
解壓縮
↓
處理資料
↓
處理完丟掉
```

---

## 3.3 emptyDir 腦中模型

```text
Node
└─ kubelet 管理的 Pod 目錄
   └─ emptyDir

Pod
├─ Container A  → 掛載 emptyDir 到 /data
├─ Container B  → 掛載 emptyDir 到 /logs
└─ emptyDir
```

重點：

```text
資料不是在 Container 裡。
資料在 Pod 對應的 Volume 目錄裡。
Container 只是掛載它。
```

---

## 3.4 emptyDir 生命週期

### Container 重啟

```text
Pod
├─ nginx Container Crash
└─ emptyDir
```

Kubelet 做的事：

```text
重新建立 nginx Container
```

結果：

```text
emptyDir 還在
資料還在
```

原因：

```text
Pod 沒死。
emptyDir 屬於 Pod。
所以 emptyDir 沒死。
```

---

### Pod 被刪除

```text
kubectl delete pod demo
```

結果：

```text
Pod 消失
↓
emptyDir 消失
↓
資料消失
```

一句話：

```text
Container 重啟，emptyDir 還在。
Pod 消失，emptyDir 消失。
```

---

## 3.5 emptyDir 真正放在哪裡

在 Linux Node 上，通常可以在 kubelet 的 Pod 目錄底下看到。

常見路徑：

```text
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>
```

例如：

```text
/var/lib/kubelet/pods/c142e124-5e2c-4464-92d8-9e7597208ed9/volumes/kubernetes.io~empty-dir/logs-volume
```

這能解釋為什麼 Container 重啟資料還在：

```text
Container 死掉
↓
Node 上的 emptyDir 目錄還在
↓
新 Container 再掛回去
↓
資料還在
```

---

## 3.6 多 Container 共用 emptyDir

### 架構

```text
Pod
├─ myapp
│  └─ /usr/local/nginx/logs  → emptyDir
├─ busybox
│  └─ /logs                 → same emptyDir
└─ emptyDir: logs-volume
```

myapp 寫：

```text
/usr/local/nginx/logs/access.log
```

busybox 看：

```text
/logs/access.log
```

雖然兩個 Container 內看到的路徑不同，但底層是同一個 emptyDir。

所以：

```text
myapp 寫入
↓
emptyDir 保存
↓
busybox 看得到
```

---

## 3.7 emptyDir Disk 版

最一般的 emptyDir：

```yaml
volumes:
  - name: logs-volume
    emptyDir: {}
```

資料會放在 Node 的磁碟上。

| 項目 | 說明 |
|---|---|
| 速度 | 一般磁碟速度 |
| 容量 | 受 Node 磁碟與 ephemeral-storage 限制 |
| Pod 刪除 | 資料消失 |
| Container 重啟 | 資料保留 |

---

## 3.8 emptyDir Memory 版

YAML：

```yaml
volumes:
  - name: mem-volume
    emptyDir:
      medium: Memory
      sizeLimit: 500Mi
```

模型：

```text
emptyDir
↓
tmpfs
↓
Node RAM
```

用途：

```text
高 I/O
低延遲
小量暫存
```

注意：

```text
Memory emptyDir 用的是記憶體。
不要拿來放大量資料。
```

---

## 3.9 emptyDir 適合什麼

| 場景 | 適不適合 | 原因 |
|---|---|---|
| 暫存檔 | 適合 | Pod 消失就丟掉沒關係 |
| Cache | 適合 | 可重建 |
| Log sidecar 共用 | 適合 | App 寫 log，Sidecar 讀 log |
| 中間運算結果 | 適合 | 任務完成就不需要 |
| 解壓縮、轉檔 | 適合 | 工作目錄 |

---

## 3.10 emptyDir 不適合什麼

不要放：

```text
MySQL 資料
PostgreSQL 資料
MongoDB 資料
Redis 持久化資料
正式交易資料
不可重建檔案
```

原因：

```text
Pod 被刪除
↓
emptyDir 被刪除
↓
資料消失
```

---

## 3.11 emptyDir 一句話

```text
emptyDir 是 Pod 專屬暫存資料夾。

Container 重啟，資料還在。
Pod 消失，資料消失。
```

---

# 4. hostPath

## 4.1 hostPath 是什麼

hostPath 是把 Node 上的實際路徑掛進 Pod。

模型：

```text
Node
└─ /data

Pod
└─ Container
   └─ /app/data  → Node:/data
```

YAML 概念：

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /data
      type: DirectoryOrCreate
```

---

## 4.2 hostPath 想解決什麼問題

emptyDir 的問題是：

```text
Pod 消失
↓
資料消失
```

hostPath 想做的是：

```text
資料不要跟著 Pod 消失。
資料直接放在 Node 上。
```

所以：

```text
Pod 刪除
↓
Node:/data 還在
↓
資料還在
```

---

## 4.3 hostPath 和 emptyDir 的差異

| 項目 | emptyDir | hostPath |
|---|---|---|
| 資料位置 | kubelet 幫 Pod 建的暫存目錄 | Node 上指定目錄 |
| Pod 刪除後 | 消失 | 通常保留 |
| 是否綁 Node | 是，Pod 期間在該 Node | 強烈綁 Node |
| 用途 | 暫存、共用 | 存取 Node 檔案、測試、特殊需求 |
| 正式資料庫 | 不適合 | 大多也不適合 |

---

## 4.4 hostPath 最大問題：Node 綁定

這是 hostPath 最重要的問題。

假設 Pod 第一次跑在 Node A：

```text
Node A
└─ /data/db
   └─ data.db

Pod
└─ 掛載 Node A:/data/db
```

Pod 重建後被調度到 Node B：

```text
Node B
└─ /data/db
```

這時 Pod 掛到的是：

```text
Node B:/data/db
```

不是：

```text
Node A:/data/db
```

所以你會看到：

```text
資料不見了？
```

但其實不是不見，是你換到另一台 Node，看的是另一個目錄。

一句話：

```text
hostPath 的資料跟 Node 綁在一起，不跟 Pod 走。
```

---

## 4.5 hostPath 為什麼危險

hostPath 讓 Container 可以碰到 Node 的檔案。

如果掛錯路徑，例如：

```text
/
/etc
/var/lib/kubelet
/var/lib/containers
/var/lib/docker
```

可能造成：

```text
Container 影響 Node
Container 讀到敏感檔案
Container 改壞主機檔案
```

所以正式環境要非常小心。

在 OpenShift / OCP 中，hostPath 通常會受到 SCC / SecurityContext 限制，不是一般應用想掛就能掛。

---

## 4.6 hostPath 適合什麼

| 場景 | 說明 |
|---|---|
| Node agent | 例如需要讀 Node log、system path |
| 監控類 DaemonSet | 每個 Node 跑一個 agent，讀該 Node 資料 |
| 測試環境 | 快速驗證掛載概念 |
| 特殊系統用途 | 例如 CSI、CNI、node-level component |

---

## 4.7 hostPath 不適合什麼

不適合一般應用保存正式資料。

原因：

```text
Pod 可能被調度到別的 Node。
hostPath 資料不會跟著 Pod 搬。
```

更合理的正式做法：

```text
PV / PVC
StorageClass
NFS
Ceph
雲端 Block Storage
```

---

## 4.8 hostPath 一句話

```text
hostPath 是把 Node 的路徑掛進 Pod。

資料跟 Node 走，不跟 Pod 走。

Pod 換 Node，就可能看到另一份資料。
```

---

# 5. 從 hostPath 到 PV / PVC

## 5.1 為什麼 hostPath 還不夠

hostPath 解決了：

```text
Pod 刪除後資料保留
```

但帶來新問題：

```text
資料綁在某一台 Node。
Pod 換 Node 就不一定拿得到原資料。
```

這對正式服務很麻煩。

例如資料庫：

```text
今天跑 Node A
明天跑 Node B
資料不能跟著過去
```

所以 Kubernetes 需要更抽象的 Storage 管理方式。

這就是 PV / PVC。

---

## 5.2 PV / PVC 要解決什麼

PV / PVC 解決的是：

```text
應用不應該直接知道底層儲存在哪裡。
```

應用只應該說：

```text
我要 10Gi
我要 ReadWriteOnce
我要某種 StorageClass
```

至於底層是：

```text
NFS
Ceph
iSCSI
雲端硬碟
本地儲存
```

應用不需要直接處理。

---

# 6. PV：PersistentVolume

## 6.1 PV 是什麼

PV 是叢集級的儲存資源。

腦中模型：

```text
Kubernetes Cluster
├─ Node
├─ Pod
└─ PV
   └─ 封裝底層儲存
```

PV 不屬於某個 Pod。

PV 的生命週期獨立於 Pod。

也就是：

```text
Pod 刪掉
↓
PV 不一定刪掉
↓
資料不一定刪掉
```

---

## 6.2 PV 是誰建立的

傳統靜態模式：

```text
Admin / 維運人員建立 PV
```

例如維運先準備 NFS：

```text
NFS Server:/nfsdata/1
```

再封裝成 PV：

```text
PV
└─ NFS Server:/nfsdata/1
```

---

## 6.3 PV 裡面定義什麼

PV 會定義：

| 欄位 | 意義 |
|---|---|
| capacity | 容量，例如 10Gi |
| accessModes | 存取模式，例如 RWO / RWX |
| persistentVolumeReclaimPolicy | PVC 刪掉後怎麼處理 |
| storageClassName | StorageClass 名稱 |
| volume source | 真正底層儲存，例如 nfs、csi |

---

## 6.4 PV 腦中模型

```text
PV
├─ 容量：10Gi
├─ 存取模式：ReadWriteOnce
├─ 回收策略：Retain
└─ 真正儲存：NFS / Ceph / Cloud Disk / Local
```

一句話：

```text
PV 是 Kubernetes 裡的儲存資源物件。
它不是資料本身。
它是對底層儲存的描述與封裝。
```

---

# 7. PVC：PersistentVolumeClaim

## 7.1 PVC 是什麼

PVC 是使用者對儲存資源的申請單。

腦中模型：

```text
使用者：
我要一顆 10Gi 的硬碟。
我要 ReadWriteOnce。
我要 storageClassName=xxx。
```

寫成 Kubernetes 物件，就是 PVC。

---

## 7.2 PVC 是誰建立的

通常是應用開發者、部署者、Helm Chart、Operator 建立。

不是每次都由 Admin 手動建立。

在 OCP 裡，很多應用會直接附 PVC，例如：

```text
資料庫
Registry
Logging
Monitoring
StatefulSet
```

---

## 7.3 PVC 不直接知道底層儲存

PVC 只說需求。

例如：

```yaml
resources:
  requests:
    storage: 10Gi
accessModes:
  - ReadWriteOnce
```

PVC 不應該關心：

```text
底層是 NFS？
底層是 Ceph？
底層是雲端 Disk？
```

這些交給 PV / StorageClass。

---

## 7.4 PVC 腦中模型

```text
PVC
├─ 我要容量：10Gi
├─ 我要模式：ReadWriteOnce
└─ 我要類型：fast / standard / nfs
```

一句話：

```text
PVC 是申請單。
PV 是可被申請的儲存資源。
```

---

# 8. PV / PVC 的關係

## 8.1 最重要圖

```text
PV（資源池）
   ↑↓ 綁定
PVC（申請單）
   ↑↓ 掛載
Pod（業務應用）
```

更口語：

```text
維運準備硬碟 = PV
使用者填申請單 = PVC
Pod 拿申請單使用硬碟 = Pod 掛 PVC
```

---

## 8.2 Binding：PVC 怎麼找到 PV

PVC 建立後，Kubernetes 控制迴圈會找符合條件的 PV。

符合條件通常看：

| 條件 | 說明 |
|---|---|
| 容量 | PV 容量要 >= PVC 要求 |
| accessModes | 存取模式要符合 |
| storageClassName | StorageClass 要符合 |
| volumeMode | Filesystem / Block 要符合 |
| selector | 如果 PVC 有 selector，PV label 要符合 |

---

## 8.3 容量匹配

PVC 要：

```text
10Gi
```

PV 有：

```text
20Gi
```

可以綁。

但結果不是切一半。

而是：

```text
整個 PV 綁給這個 PVC。
```

PV / PVC binding 通常是一對一。

---

## 8.4 Access Mode

常見模式：

| 模式 | 全名 | 直覺理解 |
|---|---|---|
| RWO | ReadWriteOnce | 可被一個 Node 讀寫掛載 |
| ROX | ReadOnlyMany | 多個 Node 唯讀掛載 |
| RWX | ReadWriteMany | 多個 Node 讀寫掛載 |
| RWOP | ReadWriteOncePod | 只能被一個 Pod 讀寫掛載 |

注意：

```text
AccessMode 是儲存後端能力，不是 Kubernetes 魔法。
```

例如：

```text
NFS 通常可支援 RWX。
很多雲端 block disk 通常偏 RWO。
```

---

## 8.5 Pod 怎麼使用 PVC

Pod 不直接掛 PV。

Pod 掛 PVC。

模型：

```text
Pod
└─ volumes:
   └─ persistentVolumeClaim:
      └─ claimName: my-pvc
```

最小概念 YAML：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc

  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
```

記法：

```text
Pod → PVC → PV → 真正儲存
```

不要記成 Pod 直接找 PV。

---

# 9. PV 回收策略

當 PVC 被刪除後，PV 怎麼辦？

這由 PV 的 reclaimPolicy 決定。

| 策略 | 意義 | 實務理解 |
|---|---|---|
| Retain | 保留資料，需要人工處理 | 安全，避免誤刪資料 |
| Delete | 刪除 PV 與底層儲存資產 | 動態供應常見 |
| Recycle | 清空再回收 | 已 deprecated，不建議使用 |

重要校正：

```text
Recycle 已被 Kubernetes 官方標示為 deprecated。
現在通常改用動態供應或手動清理流程。
```

---

## 9.1 Retain

PVC 刪掉後：

```text
PVC 刪除
↓
PV 變 Released
↓
底層資料還在
↓
Admin 手動處理
```

適合：

```text
重要資料
資料庫
不想誤刪底層資料
```

---

## 9.2 Delete

PVC 刪掉後：

```text
PVC 刪除
↓
PV 刪除
↓
底層儲存也可能被刪除
```

常見於動態建立的儲存。

注意：

```text
Delete 可能真的刪掉底層 disk / volume。
正式環境要確認 StorageClass reclaimPolicy。
```

---

## 9.3 Recycle

舊教材常會寫：

```text
Recycle = rm -rf /thevolume/*
```

但你現在要記：

```text
Recycle 已 deprecated。
不要把它當新環境建議做法。
```

---

# 10. PV 狀態

PV 常見狀態：

| 狀態 | 意義 |
|---|---|
| Available | 可用，還沒被 PVC 綁定 |
| Bound | 已被 PVC 綁定 |
| Released | PVC 已刪，但 PV 尚未被回收 |
| Failed | 自動回收失敗 |

腦中流程：

```text
Available
↓
Bound
↓
Released
↓
人工處理 / Delete / 重新建立
```

最容易搞混：

```text
Released 不代表可直接給下一個 PVC 用。
```

Released 通常代表：

```text
上一個 PVC 不在了
但資料可能還在
Kubernetes 不會直接假裝它是乾淨硬碟
```

---

# 11. PVC Protection / PV Protection

Kubernetes 有保護機制，避免你刪掉正在被使用的 PVC / PV 造成資料問題。

概念：

```text
Pod 正在用 PVC
↓
你刪 PVC
↓
PVC 不會立刻消失
↓
等 Pod 不再使用後才真正刪除
```

你可能看到 PVC 卡在：

```text
Terminating
```

並看到 finalizer：

```text
kubernetes.io/pvc-protection
```

PV 也有類似保護：

```text
kubernetes.io/pv-protection
```

一句話：

```text
PVC/PV Protection 是 Kubernetes 防呆。
不是壞掉。
```

---

# 12. StorageClass

## 12.1 為什麼需要 StorageClass

前面講的是靜態 PV：

```text
Admin 先建 PV
User 再建 PVC
PVC 找 PV
```

但這很麻煩。

如果每個 PVC 都要 Admin 手動先建 PV，維運會爆炸。

所以有 StorageClass。

---

## 12.2 StorageClass 是什麼

StorageClass 不是儲存空間。

StorageClass 是：

```text
動態建立 PV 的規則。
```

模型：

```text
PVC
↓
指定 storageClassName
↓
StorageClass
↓
Provisioner
↓
自動建立 PV
↓
Pod 使用
```

---

## 12.3 靜態供應 vs 動態供應

| 模式 | 流程 |
|---|---|
| 靜態供應 | Admin 先建 PV，PVC 再綁 |
| 動態供應 | User 建 PVC，StorageClass 自動建 PV |

---

## 12.4 StorageClass 腦中模型

```text
StorageClass
├─ provisioner：誰負責建立儲存
├─ parameters：建立儲存的參數
├─ reclaimPolicy：刪 PVC 後怎麼處理
├─ volumeBindingMode：何時綁定
└─ allowVolumeExpansion：是否允許擴容
```

---

## 12.5 volumeBindingMode

常見兩種：

| 模式 | 意義 |
|---|---|
| Immediate | PVC 建立後立刻建立 / 綁定 PV |
| WaitForFirstConsumer | 等 Pod 出現並知道要排到哪裡，再建立 / 綁定 PV |

為什麼需要 WaitForFirstConsumer？

因為有些儲存跟 zone / node 有關。

如果太早建立 PV，可能建立在錯的區域，Pod 排程會卡住。

---

# 13. NFS + PV / PVC

## 13.1 NFS 在這裡扮演什麼角色

NFS 是真正放資料的地方。

模型：

```text
NFS Server
└─ /nfsdata/1

Kubernetes
└─ PV
   └─ 指向 NFS:/nfsdata/1

PVC
└─ 申請 PV

Pod
└─ 掛載 PVC
```

一句話：

```text
NFS 是底層儲存。
PV 是 Kubernetes 對 NFS 的封裝。
PVC 是使用者對 PV 的申請。
Pod 是實際使用者。
```

---

## 13.2 NFS 的好處

| 好處 | 說明 |
|---|---|
| 可跨 Node | Pod 在不同 Node 仍可掛同一份 NFS |
| 可 RWX | 適合多 Pod 共享檔案 |
| 好理解 | 比 Ceph / CSI 容易入門 |

---

## 13.3 NFS 的限制

| 限制 | 說明 |
|---|---|
| 效能 | 取決於 NFS Server 與網路 |
| 單點 | 單一 NFS Server 可能是 SPOF |
| 權限 | UID/GID、root_squash/no_root_squash 要小心 |
| 安全 | 不要隨便 `*` 開給所有來源 |
| 資料庫 | 大型 DB 通常不建議直接放一般 NFS |

---

## 13.4 為什麼教材會叫每個 Node 裝 nfs-utils

因為 Pod 可能被排到任一 Worker Node。

如果該 Node 要掛 NFS，它需要 NFS client 工具與 kernel 支援。

所以：

```text
Pod 在 node01
↓
node01 要能掛 NFS

Pod 在 node02
↓
node02 也要能掛 NFS
```

這不是每個 Node 都要有資料。

而是每個 Node 都要有能力連到 NFS。

---

## 13.5 NFS 跟 hostPath 最大差異

| 項目 | hostPath | NFS |
|---|---|---|
| 資料在哪 | 某一台 Node | NFS Server |
| Pod 換 Node | 可能看到不同資料 | 還是同一份 NFS 資料 |
| 適合多 Node | 不適合 | 較適合 |
| RWX | 不是真正跨 Node RWX | 通常可 RWX |

一句話：

```text
hostPath 是看每台 Node 自己的目錄。
NFS 是大家都連去同一台檔案伺服器。
```

---

# 14. OpenShift / OCP 實務觀念

## 14.1 在 OCP 不要亂用 hostPath

OCP 比一般 Kubernetes 更重視安全限制。

hostPath 代表 Pod 可以碰 Node 檔案。

所以在 OCP：

```text
一般 workload 不應該隨便用 hostPath。
```

常見會用 hostPath 的通常是：

```text
DaemonSet
Node agent
CSI / CNI / Logging agent
需要特權的系統元件
```

一般業務應用要持久化資料，優先看：

```text
PVC
StorageClass
Operator 提供的 storage 設定
```

---

## 14.2 OCP 中 Ready 不等於 Storage 正常

Pod Ready 只代表 readiness probe 通過。

不代表：

```text
PVC 一定正常
資料一定有寫入
底層 Storage 一定健康
```

要看 Storage，要另外查：

```bash
oc get pvc -A
oc get pv
oc describe pvc <pvc>
oc describe pv <pv>
oc get storageclass
```

如果是 CSI，還要看 CSI driver / provisioner 狀態。

---

## 14.3 OCP 實務排查順序

### 第一步：看 PVC

```bash
oc get pvc -A
```

目的：

```text
確認 PVC 是 Bound 還是 Pending。
```

結果判斷：

| 狀態 | 意義 |
|---|---|
| Bound | PVC 已綁到 PV |
| Pending | 沒找到合適 PV，或 StorageClass/provisioner 有問題 |
| Terminating | 可能還被 Pod 使用或 finalizer 卡住 |

---

### 第二步：看 Pod 掛載事件

```bash
oc describe pod <pod> -n <namespace>
```

目的：

```text
看 Events 裡有沒有 MountVolume、AttachVolume、permission denied、timeout。
```

---

### 第三步：看 StorageClass

```bash
oc get storageclass
```

目的：

```text
確認 PVC 指定的 storageClassName 是否存在。
```

---

### 第四步：看 PV

```bash
oc get pv
```

目的：

```text
確認 PV 狀態、容量、AccessMode、ReclaimPolicy。
```

---

## 14.4 OCP Storage 一句話

```text
業務應用要資料持久化：
優先 PVC。

不要把 hostPath 當正式儲存。

emptyDir 只當暫存。

StorageClass 是動態建 PV 的規則。
```

---

# 15. 最容易搞混的觀念整理

## 15.1 Container 重啟 vs Pod 重建

```text
Container 重啟：
Pod 還在，Volume 多半還在。

Pod 重建：
舊 Pod 消失，新 Pod 是另一個物件。
```

---

## 15.2 emptyDir vs hostPath

```text
emptyDir：
Kubelet 幫 Pod 建暫存目錄。
Pod 消失，資料消失。

hostPath：
直接用 Node 路徑。
Pod 消失，Node 上資料還在。
但 Pod 換 Node，就看另一台 Node 的路徑。
```

---

## 15.3 PV vs PVC

```text
PV：
叢集裡已存在或被動態建立的儲存資源。

PVC：
使用者提出的儲存申請。
```

---

## 15.4 Pod 用的是 PVC，不是 PV

```text
Pod
↓
PVC
↓
PV
↓
底層儲存
```

不要記反。

---

## 15.5 StorageClass 不是硬碟

```text
StorageClass 是規則。
不是資料存放地。
```

它負責告訴 Kubernetes：

```text
PVC 來了，要找哪個 provisioner 建 PV。
```

---

## 15.6 NFS 不是 PV

```text
NFS 是底層儲存。
PV 是 Kubernetes 物件。
PVC 是申請單。
Pod 是使用者。
```

---

# 16. 最後總結

如果你只想記一頁，就記這個：

```text
Volume：
把資料從 Container 抽離出來。

emptyDir：
Pod 專屬暫存資料夾。
Container 重啟資料還在。
Pod 消失資料消失。

hostPath：
把 Node 路徑掛進 Pod。
資料跟 Node 走，不跟 Pod 走。

PV：
Kubernetes 裡的儲存資源。

PVC：
使用者對儲存資源的申請單。

Pod：
不直接用 PV。
Pod 掛 PVC。

StorageClass：
動態建立 PV 的規則。

NFS：
真正放資料的檔案伺服器。
PV 可以封裝 NFS 給 PVC 使用。

OCP：
正式應用優先 PVC / StorageClass。
emptyDir 當暫存。
hostPath 少用，因為安全與 Node 綁定風險高。
```
