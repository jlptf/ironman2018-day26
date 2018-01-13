## Day 26 - 部署多階層應用：SAB

### 本日共賞
* 簡易通訊錄架構
* 部署簡易通訊錄

### 希望你知道

* [k8s 不自賞 - 基礎篇](https://ithelp.ithome.com.tw/users/20107062/ironman/1244)

<br/>

今天我們來部署一個簡易通訊錄 (Simple Address Book, SAB)。

#### 簡易通訊錄架構

經過這麼多天，相信大家對 k8s 的物件都有所認識。接下來我們來實際部署一個簡易通訊錄應用程式 (`SAB`)，在討論架構之前，讓我們先看看程式實際運作時候的畫面

![](https://ithelp.ithome.com.tw/upload/images/20180105/20107062zsUUHZpTNG.png)

`SAB` 紀錄 `暱稱` 與 `Email` 兩個欄位，並使用 [mongodb](https://www.mongodb.com/download-center) 儲存資料。當使用者新增內容，程式會先檢查資料是否已經存在，如果不存在則會寫入資料庫並更新目前通訊錄資料。下圖顯示欲新增的資料已存在：

![](https://ithelp.ithome.com.tw/upload/images/20180105/20107062AFVidHuJTy.png)

> 這只是簡易的測試程式，所以並沒有實作刪除功能

大家應該有看到畫面中有一個 `IP Address` 的欄位，這個欄位會顯示目前容器的 IP，可以順便檢視當 k8s 部署應用程式在多個 Pod 的狀況。

大致上瞭解了這個程式的功用，接下來我們來看一下 `SAB` 架構

![](https://ithelp.ithome.com.tw/upload/images/20180105/20107062s88MbsR36l.png)

> 為簡易說明，這裡只使用 Service 物件並未使用 Ingress 物件，大家應該還記得 [Ingress](https://ithelp.ithome.com.tw/articles/10193528) 物件吧？

`SAB` 內有兩層架構，一個是應用程式本身 `app: address-book`，另外一個是資料庫 `app: mongodb`。這裡只對外提供一個 Service `addressbook-svc`，而資料庫是藏在叢集中，外部是無法存取的。因此，如果要修改資料庫，只能透過 `app: address-book`。上圖中有兩個 Pod 用虛線表示，意思是隨時可視需要增加或減少。

<br/>

#### 部署簡易通訊錄

底下是 `mongodb.yaml` 的內容

```
# mongodb.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
  labels:
    app: mongodb    <=== 添加標記 app: mongodb，以便綁定 Service
spec:
  containers:
  - name: mongodb
    image: mongo    <=== mongo 官方映像檔
    ports:
      - containerPort: 27017  <=== mongodb 預設 port 為 27017

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc  <=== 宣告 Service 名稱
spec:
  type: ClusterIP    <=== 只需要叢集內部存取，使用 ClusterIP 即可
  selector:
    app: mongodb     <=== 綁定 label 為 app: mongodb 的 Pod
  ports:
    - protocol: TCP
      port: 27017
```

在這裡，我們準備部署一個 Pod 專門運行 Mongo 資料庫。這裡沒有太特別的的設定，只需要注意映像的名稱與 Port 即可部署到 k8s

```bash
$ kubectl apply -f mongodb.yaml
pod "mongodb" created
service "mongodb-svc" created

$ kubectl get service
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP     2d
mongodb-svc   ClusterIP   10.103.195.164   <none>        27017/TCP   1m
```

> Service 名稱 `mongodb-svc` 很重要，我們需要在 Pod 中綁定這個名稱，讓Pod 能夠知道有這個 Service 存在，才能夠順利存取資料庫

接著看一下 `address-book` 的內容

```
# addressbook.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: address-book
spec:
  replicas: 1    <=== 先部署一個 Pod，之後再透過指令變更
  template:
    metadata:
      labels: 
        app: address-book   <=== 添加標記 app: address-book，以便綁定 Service
    spec:
      containers:
      - name: address-book
        image: jlptf/addressbook   <=== 預先建立好的映像檔
        env:
        - name: SERVER_PORT    <=== 透過環境變數指定容器 port
          value: "80"
        - name: DB_SERVER      <=== 透過環境變數指定資料庫位置
          value: "mongodb-svc"
        ports:
        - containerPort: 80
    
---
apiVersion: v1
kind: Service
metadata:
  name: addressbook-svc   
spec: 
  type: NodePort    <=== NodePort 型態供外部存取
  selector:
    app: address-book   <=== 綁定 label 有 app: address-book 的 Pod
  ports:
    - protocol: TCP
      port: 80
```

`jlptf/addressbook` 是 [go language](https://golang.org/) 寫的應用程式，裡面有兩個環境變數

* `SERVER_PORT`：指定應用程式的 Port 
* `DB_SERVER`：指定資料庫的位置

> 大家可以試試看用 ConfigMap 的方式設定環境變數，另外，對 go 有興趣的朋友可以參考這次鐵人賽的文章：
> 
> [網襪工程師](https://ithelp.ithome.com.tw/users/20107302/profile)：[Go！從無到打造最佳行動網站](https://ithelp.ithome.com.tw/users/20107302/ironman/1242)

上面我們已經部署了 Mongo 資料庫，並與名為 `mongodb-svc` 的 Service 綁定，故 `DB_SERVER` 只需要設成該名稱即 `mongodb-svc`。部署到 k8s

```bash
$ kubectl apply -f addressbook.yaml
deployment "address-book" created
service "addressbook-svc" created

$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
address-book-5766864845-xznt2   1/1       Running   0          11s
mongodb                         1/1       Running   0          6m

$ kubectl get service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
addressbook-svc   NodePort    10.100.123.38    <none>        80:30554/TCP   1m
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        2d
mongodb-svc       ClusterIP   10.103.195.164   <none>        27017/TCP      7m
```

> 這裡可能要等一下，因為要下載映像檔 

到這裡聰明的大家應該知道要怎麼存取 `SAB` 了吧？k8s 幫 addressbook-svc 綁定了一個 `PORT:30554`，我們可以在瀏覽器鍵入 `http://[minikube ip]:30554` 就可以看到 `SAB`，大家可以試試新增資料以及新增一個重複的資料。

> 忘了什麼是 minikube ip 了嗎？可以參考文章 [Day 5 - 運行 minikube](https://ithelp.ithome.com.tw/articles/10193237)
> 
> 忘記什麼是 Service 了嗎？可以參考文章 [Day 11 - Service (1)](https://ithelp.ithome.com.tw/articles/10193520)
> 
> 不知道我在搞什麼東西嗎？來 [k8s 不自賞](https://ithelp.ithome.com.tw/users/20107062/ironman/1244) 可以幫助你

接下來，我們試著讓應用程式 `長大`，輸入以下指令

```bash
$ kubectl scale --replicas=3 deployment/address-book
deployment "address-book" scaled

$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
address-book-5766864845-stth4   1/1       Running   0          31s
address-book-5766864845-xznt2   1/1       Running   0          17m
address-book-5766864845-zg2vr   1/1       Running   0          31s
mongodb                         1/1       Running   0          24m
```

這時候你會發現 `SAB` 運行在三個不同的 Pod 上，接著你可以開啟 `SAB` 頁面並試著刷新頁面 `Command + R`，你會看到畫面中的 `IP Address` 會有變化，表示你的需求是會被導到不同的 Pod 處理的。

![https://ithelp.ithome.com.tw/upload/images/20180105/20107062qnmQrxMPIL.png](https://ithelp.ithome.com.tw/upload/images/20180105/20107062qnmQrxMPIL.png)

這呼應到我們在第一天提到的 Container Orchestrator 的好處

* `可容錯`：多個 Pod 都提供相同的服務，當其中某些 Pod 發生問題，還有別的 Pod 能處理
* `可擴展`：你剛剛不是 `長大` 了嗎？

大家有沒有發現有個問題？如果資料庫發生問題，Pod 需要重新建立的時候，資料就不見了！不用擔心，我們來修正一下 `mongodb.yaml`，多掛載一個 Volume 給他。

> 或者你可以把資料庫的 Pod 直接刪除讓 k8s 重新建立，就會發現原本的資料都不見了

```
# mongodb.modify.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  containers:
  - name: mongodb
    image: mongo
    ports:
      - containerPort: 27017
    volumeMounts:           <=== 這裡開始是新增加的內容，將 Volume 掛載到 /data/db 目錄
    - mountPath: /data/db
      name: mongo-data
  volumes:                  <=== 指定目錄 /tmp/mongo 放資料
  - name: mongo-data
    hostPath:
      path: /tmp/mongo      <=== 到這裡是新增加的內容

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
```

這裡我們幫 mongo 掛載了一個 Volume，`/data/db` 是資料庫放置資料的位置，接著把修改過後的內容部署到 k8s

```bash
$ kubectl apply -f mongodb.modify.yaml
pod "mongodb" created
service "mongodb-svc" unchanged
```

> 記得先把之前部署的 mongodb 刪除

接著你可以新增資料後，把 `mongodb` Pod 刪除，再重新部署一次，你會發現你新增的資料不會消失不見，就表示成功了！

> 這裡要提醒大家，這樣的方式只能用在 minikube 測試環境中。實務上，k8s 叢集內可能會有很多 Node，而你每次部署不見得都會在同一台 Node (雖然有辦法指定)，就算你只有一台 Node 也不會建議用這種方式。在雲端平台，你可以把資料放在雲端磁碟中，而 Pod 與雲端磁碟也是解耦的關係，需要使用磁碟再掛到 Pod 使用。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day26/](https://jlptf.github.io/ironman2018-day26/)

