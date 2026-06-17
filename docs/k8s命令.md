# kubectl create deployment（dry-run=client）

## 一句話用途
在本機端模擬建立 Deployment，輸出 YAML 範本，不會對 Kubernetes 叢集造成任何實際變更。

---

## 指令全文

    kubectl create deployment web --image=nginx --dry-run=client -o yaml

---

## 參數逐一說明

### kubectl
- Kubernetes 官方 CLI 工具
- 用來建立、查詢、修改叢集資源

### create
- 動作：建立資源
- 搭配 --dry-run=client 時，只進行模擬，不會實際建立

### deployment
- 資源種類
- 表示要產生的是 Deployment 物件

### web
- Deployment 名稱
- 對應 YAML 節點：

    metadata:
      name: web

### --image=nginx
- 指定容器映像檔
- kubectl 會自動補齊 selector 與 labels

    containers:
    - name: nginx
      image: nginx

### --dry-run=client
- 僅在本機端計算
- 不送 API Server
- 不寫入 etcd
- 不建立任何實體資源

### -o yaml
- 指定輸出格式為 YAML
- 用來產生可編輯的資源範本

---

## 常見實務用法

### 產生 YAML 檔案

    kubectl create deployment test_web --image=nginx --dry-run=client -o yaml > test_web.yaml

### 套用到叢集

    kubectl apply -f test_web.yaml

---

## 範例輸出（dry-run 結果）

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: test_web
      labels:
        app: test_web
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: test_web
      template:
        metadata:
          labels:
            app: test_web
        spec:
          containers:
          - name: nginx
            image: nginx
            resources: {}
    status: {}

---

## 重點總結

kubectl create  
+ --dry-run=client  
+ -o yaml  

等於：  
由 kubectl 在本機端產生一份合法、可直接使用的 Deployment YAML 範本
