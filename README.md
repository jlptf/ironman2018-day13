## Day 13 - Ingress

### 本日共賞

* Ingress

### 希望你知道
* [Service(1)](https://ithelp.ithome.com.tw/articles/10193520)
* [Service(2)](https://ithelp.ithome.com.tw/articles/10193527)

<br/>

#### Ingress

昨天我們已經可以透過 Service 物件連到 Pod，接下來的問題就如同 [Day 11 - Service(1)](https://ithelp.ithome.com.tw/articles/10193520) 提到的如何存取 Pod 問題類似：難道我們需要使用者知道每個 Node 的 IP 嗎？

我們成功地利用 Service 把存取 Pod 的工作抽象化，即把存取 Pod 的工作轉換成存取 Service，我們可以透過 Node IP:Port 的方式存取 Pod，但是在 k8s 中 Node 是可以視需求隨時增加或減少的，也因此 Node IP 隨時有可能會有變動，因此 Ingress 物件就是再一次幫我們把存取 Service 抽象化的好東西。

我們用一張圖來描述

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062tOO6jxwuvR.png)

這裡要發揮一下想像力，想像一下整個 k8s 叢集由一群 Node 組成並提供多個不同的服務 (Service)，就像圖中的四個不同顏色的 Service，我們希望使用者能夠從一個單一的入口就能夠存取 k8s 中不同的服務，而 Ingress 物件就能幫我們達到這樣的目的。

底下我們來實作一下，先看個圖

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062BgP085nOMv.png)

如上圖，我們希望在 k8s 中建立兩個 Service 分別是 `red-service` 與 `green-service`。

`red-service ` 綁定四個標記為 `app: red-nginx` 的 Pod 而 `green-service ` 綁定兩個標記為 `app: green-nginx` 的 Pod。使用者可以透過 `http://green.myweb.com` 與 `http://red.myweb.com` 來分別瀏覽 `green-service` 與 `red-service`。

讓我們先看看 Deployment 的內容

```
# ingress.deployment.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: red-nginx
spec:
    replicas: 4   <=== 我們希望 red-service 有四個 Pod 運行 red-nginx
    template:
      metadata:
        labels: 
          app: red-nginx
      spec:
        containers:
        - name: nginx
          image: jlptf/red   <=== 預先做好的 Docker 映像檔
          ports:
          - containerPort: 80

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: green-nginx
spec:
    replicas: 2   <=== 我們希望 green-service 有兩個 Pod 運行 green-nginx
    template:
      metadata:
        labels: 
          app: green-nginx
      spec:
        containers:
        - name: nginx
          image: jlptf/green   <=== 預先做好的 Docker 映像檔
          ports:
          - containerPort: 80
```

部署到 k8s 並觀察

```bash
$ kubectl apply -f ingress.deployment.yaml 
deployment "red-nginx" created
deployment "green-nginx" created

$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
green-nginx-f5989dfd9-fcm5q   1/1       Running   0          47s
green-nginx-f5989dfd9-hzbq5   1/1       Running   0          47s
red-nginx-787f484d9c-grbsk    1/1       Running   0          34s
red-nginx-787f484d9c-h7pl6    1/1       Running   0          47s
red-nginx-787f484d9c-m897h    1/1       Running   0          29s
red-nginx-787f484d9c-n946g    1/1       Running   0          47s
```

很順利，運行了六個 Pod 預計分別給 `red-service` 與 `green-service` 使用，接著看看 Service 的內容

> 看 prefix 應該不難分辨哪個 Pod 是給哪個 Service 使用的吧！所以說名字取的好，爸爸沒煩惱！是吧～～

```
# ingress.service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: green-service
spec:
  type: NodePort
  selector:
    app: green-nginx   <=== 綁定標記為 green-nginx 的 Pod
  ports:
    - protocol: TCP
      port: 80

---
apiVersion: v1
kind: Service
metadata:
  name: red-service
spec:
  type: NodePort
  selector:
    app: red-nginx   <=== 綁定標記為 red-nginx 的 Pod
  ports:
    - protocol: TCP
      port: 80
```

> 經過這幾天的練習，上面的 yaml 應該不難閱讀了吧

同樣部署到 k8s 並觀察狀態

```bash
$ kubectl apply -f ingress.service.yaml
service "green-service" created
service "red-service" created

$ kubectl get services
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
green-service   NodePort    10.0.0.133   <none>        80:30615/TCP   16s
kubernetes      ClusterIP   10.0.0.1     <none>        443/TCP        15d
red-service     NodePort    10.0.0.75    <none>        80:32101/TCP   16s
```

然後是今天的主角 Ingress

```
# ingress.yaml

---
apiVersion: extensions/v1beta1
kind: Ingress   <=== 指定物件種類為 Ingress
metadata:
  name: web
spec:
  rules:
  - host: green.myweb.com   <=== 指定 host 名稱為 green.myweb.com
    http:
      paths:
      - backend:
          serviceName: green-service  <=== 綁定名稱為 green-service 的 Service 物件
          servicePort: 80
  - host: red.myweb.com   <=== 指定另一個 host 名稱為 red.myweb.com
    http:
      paths:
      - backend:
          serviceName: red-service   <=== 綁定名稱為 red-service 的 Service 物件
          servicePort: 80
```

類似 Service 綁定 Pod 的概念，Ingress 透過 `serviceName` 指定需要綁定的 Service。接著部署到 k8s

```bash
$ kubectl apply -f ingress.yaml 
ingress "web" created

$ kubectl get ingress
NAME      HOSTS                           ADDRESS          PORTS     AGE
web       green.myweb.com,red.myweb.com   192.168.99.100   80        43s
```

這裡就會看到 `green.myweb.com` 與 `red.myweb.com` 都被綁定到 `192.168.99.100` 的位置

> 如果在雲端提供商上建立 Ingress，你可以在 `ADDRESS` 看到被分配的 IP

由於 `green.myweb.com` 與 `red.myweb.com` 這兩個網址並非真正存在，因此，如果你在瀏覽器上打上這兩個網址，並不會有任何內容。

我們要讓自己的電腦能夠理解這兩個網址的 IP 位置，在 Mac/Linux 環境下可以將以下內容加到 `/etc/hosts` 內

> Windows 請修改 `\WINDOWS\system32\drivers\etc\hosts`

```
192.168.99.100   green.myweb.com  <=== 192.168.99.100 是 Ingress 分配的 IP
192.168.99.100   red.myweb.com
```

你可以使用以下指令將上面的內容加到 `/etc/hosts` 內

```bash
$ echo 192.168.99.100   green.myweb.com >> /etc/hosts
$ echo 192.168.99.100   red.myweb.com >> /etc/hosts
```

> 你也可以透過 vim 編輯，可能需要 sudo 權限


打開瀏覽器並鍵入網址，一切順利的話你會看到

`http://green.myweb.com`

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062UcnLyK6qzK.png)

`http://red.myweb.com`

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062q5TQtd91xQ.png)

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day13/](https://jlptf.github.io/ironman2018-day13/)

另外，建立 `jlptf/red` 與 `jlptf/green` 這兩個映像檔用到的 `Dockerfile` 你可以在 `ingressDocker` 資料夾裡面找到，如果你想要自己動手建立映像檔可以參考使用。

建立映像檔參考指令如下：

```bash
$ docker build --build-arg HTML=green.html -t jlptf/green .

$ docker build --build-arg HTML=red.html -t jlptf/red .
```

這邊就不對 Docker 內容說明，請自行參考文章

> [yangj26952](https://ithelp.ithome.com.tw/users/20103456/profile)： [用30天來介紹和使用 Docker](https://ithelp.ithome.com.tw/users/20103456/ironman/1320)
>
> [jia_hong](https://ithelp.ithome.com.tw/users/20107537/profile)： [讓我們來玩玩Docker吧](https://ithelp.ithome.com.tw/users/20107537/ironman/1417)
