# Домашнее задание к занятию «Обновление приложений»

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

## Ответ:

Так как обновление мажорное и новые версии приложения не умеют работать со старыми(тоесть требуется менять и фронтэнд и бэкэнд), то мы не сможем  использовать A/B развертывания и канареечные развертывания.
Ресурсы ограничены, значит Blue/Green тоже не подходит.
В условии не указано возможна ли остановка сервиса, значит мы можем использовать стратегию Recreat.
Но так как небольшой запас по ресурсам все же имеется и приложение состоит из нескольких реплик, то можно рассмотреть стратегию Rolling.
Например уменьшив количество реплик в менее загруженный момент времени, провести Rolling update и по завершению, снова увеличить количество реплик до изначального.

### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.

Создал deployment и сервис сразу с учетом обновления:   
Стратегия Rolling. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-task2
  labels:
    app: app-task2
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 3
  selector:
    matchLabels:
      app: app-task2
  template:
    metadata:
      labels:
        app: app-task2
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
---
apiVersion: v1
kind: Service
metadata:
  name: app-task2
spec:
  selector:
    app: app-task2
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80
    - name: multitool-http
      protocol: TCP
      port: 8080
      targetPort: 8080

```
Запускаю и проверяю:
```bash
user@microk8s:~/kuber-homeworks-3.4$ kubectl apply -f app-task2.yaml 
deployment.apps/app-task2 created
service/app-task2 created
user@microk8s:~/kuber-homeworks-3.4$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
app-task2-557484df6-cs7pq   2/2     Running   0          9s
app-task2-557484df6-fhsn4   2/2     Running   0          9s
app-task2-557484df6-fr8ks   2/2     Running   0          9s
app-task2-557484df6-kmcv6   2/2     Running   0          9s
app-task2-557484df6-nf2bh   2/2     Running   0          9s
user@microk8s:~/kuber-homeworks-3.4$ kubectl describe deployments.apps app-task2 | grep Image
    Image:        nginx:1.19
    Image:      wbitt/network-multitool
user@microk8s:~/kuber-homeworks-3.4$ 
user@microk8s:~/kuber-homeworks-3.4$ kubectl exec deployments/app-task2 -c multitool -it -- bash
app-task2-557484df6-cs7pq:/# curl -I app-task2:80
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sat, 30 Nov 2024 03:32:09 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes

```
Меняю параметр __image: nginx:1.19__ на __image: nginx:1.20__   
Запускаю и проверяю:   
```bash 
user@microk8s:~/kuber-homeworks-3.4$ kubectl get po -w
app-task2-557484df6-7xdb9    2/2     Running             0          24m
app-task2-557484df6-qdvfw    2/2     Running             0          24m
app-task2-6f49d9b64d-5gd4l   0/2     ContainerCreating   0          5s
app-task2-6f49d9b64d-8qxqf   0/2     ContainerCreating   0          5s
app-task2-6f49d9b64d-cdscp   0/2     ContainerCreating   0          5s
app-task2-6f49d9b64d-gd57m   0/2     ContainerCreating   0          5s
app-task2-6f49d9b64d-wbbcn   0/2     ContainerCreating   0          5s
app-task2-6f49d9b64d-8qxqf   2/2     Running             0          21s
app-task2-6f49d9b64d-cdscp   2/2     Running             0          21s
app-task2-6f49d9b64d-5gd4l   2/2     Running             0          21s
app-task2-557484df6-7xdb9    2/2     Terminating         0          24m
app-task2-557484df6-qdvfw    2/2     Terminating         0          24m
app-task2-557484df6-7xdb9    2/2     Terminating         0          24m
app-task2-557484df6-qdvfw    2/2     Terminating         0          24m
app-task2-557484df6-7xdb9    0/2     Completed           0          24m
app-task2-6f49d9b64d-gd57m   2/2     Running             0          22s
app-task2-6f49d9b64d-wbbcn   2/2     Running             0          22s
app-task2-557484df6-qdvfw    0/2     Completed           0          24m
app-task2-557484df6-7xdb9    0/2     Completed           0          24m
app-task2-557484df6-7xdb9    0/2     Completed           0          24m
app-task2-557484df6-qdvfw    0/2     Completed           0          24m
app-task2-557484df6-qdvfw    0/2     Completed           0          24m
^C
user@microk8s:~/kuber-homeworks-3.4$ kubectl describe deployments.apps app-task2 | grep Image
    Image:        nginx:1.20
    Image:      wbitt/network-multitool
user@microk8s:~/kuber-homeworks-3.4$ kubectl exec deployments/app-task2 -c multitool -it -- bash
app-task2-6f49d9b64d-5mlvt:/# curl -I app-task2:80
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Sat, 30 Nov 2024 04:00:06 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Nov 2021 14:44:02 GMT
Connection: keep-alive
ETag: "6193c3b2-264"
Accept-Ranges: bytes
```

3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.   
Меняю параметр __image: nginx:1.20__ на __image: nginx:1.28__
Приложение не может обновиться:
```bash
user@microk8s:~/kuber-homeworks-3.4$ kubectl get po -w
NAME                         READY   STATUS             RESTARTS   AGE
app-task2-6f49d9b64d-5mlvt   2/2     Running            0          6m44s
app-task2-6f49d9b64d-kjwwg   2/2     Running            0          6m43s
app-task2-799677b7cb-92zvd   1/2     ImagePullBackOff   0          8s
app-task2-799677b7cb-ggm86   1/2     ErrImagePull       0          8s
app-task2-799677b7cb-kf5pj   1/2     ErrImagePull       0          8s
app-task2-799677b7cb-r45hf   1/2     ErrImagePull       0          8s
app-task2-799677b7cb-z7gfs   1/2     ErrImagePull       0          8s
app-task2-799677b7cb-kf5pj   1/2     ImagePullBackOff   0          9s
app-task2-799677b7cb-z7gfs   1/2     ImagePullBackOff   0          9s
app-task2-799677b7cb-r45hf   1/2     ImagePullBackOff   0          9s
app-task2-799677b7cb-ggm86   1/2     ImagePullBackOff   0          9s
app-task2-799677b7cb-z7gfs   1/2     ErrImagePull       0          35s
app-task2-799677b7cb-kf5pj   1/2     ErrImagePull       0          36s
app-task2-799677b7cb-r45hf   1/2     ErrImagePull       0          37s
app-task2-799677b7cb-92zvd   1/2     ErrImagePull       0          38s
app-task2-799677b7cb-ggm86   1/2     ErrImagePull       0          38s
app-task2-799677b7cb-kf5pj   1/2     ImagePullBackOff   0          47s
app-task2-799677b7cb-z7gfs   1/2     ImagePullBackOff   0          49s
app-task2-799677b7cb-r45hf   1/2     ImagePullBackOff   0          50s
app-task2-799677b7cb-ggm86   1/2     ImagePullBackOff   0          53s
app-task2-799677b7cb-92zvd   1/2     ImagePullBackOff   0          53s
app-task2-799677b7cb-kf5pj   1/2     ErrImagePull       0          64s
app-task2-799677b7cb-z7gfs   1/2     ErrImagePull       0          65s
app-task2-799677b7cb-r45hf   1/2     ErrImagePull       0          67s
app-task2-799677b7cb-ggm86   1/2     ErrImagePull       0          69s
app-task2-799677b7cb-92zvd   1/2     ErrImagePull       0          71s
app-task2-799677b7cb-kf5pj   1/2     ImagePullBackOff   0          75s
app-task2-799677b7cb-z7gfs   1/2     ImagePullBackOff   0          77s
app-task2-799677b7cb-r45hf   1/2     ImagePullBackOff   0          82s
app-task2-799677b7cb-92zvd   1/2     ImagePullBackOff   0          83s
app-task2-799677b7cb-ggm86   1/2     ImagePullBackOff   0          84s
app-task2-799677b7cb-r45hf   1/2     ErrImagePull       0          113s
app-task2-799677b7cb-kf5pj   1/2     ErrImagePull       0          114s
app-task2-799677b7cb-z7gfs   1/2     ErrImagePull       0          2m2s
app-task2-799677b7cb-92zvd   1/2     ErrImagePull       0          2m6s
app-task2-799677b7cb-kf5pj   1/2     ImagePullBackOff   0          2m6s
app-task2-799677b7cb-ggm86   1/2     ErrImagePull       0          2m6s
app-task2-799677b7cb-r45hf   1/2     ImagePullBackOff   0          2m7s
app-task2-799677b7cb-z7gfs   1/2     ImagePullBackOff   0          2m16s
app-task2-799677b7cb-92zvd   1/2     ImagePullBackOff   0          2m19s
app-task2-799677b7cb-ggm86   1/2     ImagePullBackOff   0          2m19s
app-task2-799677b7cb-r45hf   1/2     ErrImagePull       0          3m19s
app-task2-799677b7cb-kf5pj   1/2     ErrImagePull       0          3m21s
app-task2-799677b7cb-92zvd   1/2     ErrImagePull       0          3m29s
app-task2-799677b7cb-r45hf   1/2     ImagePullBackOff   0          3m32s
app-task2-799677b7cb-kf5pj   1/2     ImagePullBackOff   0          3m36s
app-task2-799677b7cb-z7gfs   1/2     ErrImagePull       0          3m39s
app-task2-799677b7cb-92zvd   1/2     ImagePullBackOff   0          3m40s
app-task2-799677b7cb-ggm86   1/2     ErrImagePull       0          3m41s

user@microk8s:~/kuber-homeworks-3.4$ kubectl describe events
Name:             app-task2-799677b7cb-92zvd.180ca3233f0a18a1
Namespace:        app
Labels:           <none>
Annotations:      <none>
API Version:      v1
Count:            44
Event Time:       <nil>
First Timestamp:  2024-11-30T04:02:08Z
Involved Object:
  API Version:       v1
  Field Path:        spec.containers{nginx}
  Kind:              Pod
  Name:              app-task2-799677b7cb-92zvd
  Namespace:         app
  Resource Version:  1400116
  UID:               fd125a56-c68b-43a9-bb5a-c7b31efbc6cf
Kind:                Event
Last Timestamp:      2024-11-30T04:12:13Z
Message:             Back-off pulling image "nginx:1.28"
Metadata:
  Creation Timestamp:  2024-11-30T04:02:08Z
  Resource Version:    1401908
  UID:                 c347a79b-dbe9-436d-a098-3cc9376ee019
Reason:                BackOff
Reporting Component:   kubelet
Reporting Instance:    microk8s
Source:
  Component:  kubelet
  Host:       microk8s
Type:         Normal
Events:       <none>
```

тем временем проверяю доступность :
```bash
user@microk8s:~$ kubectl describe deployments.apps app-task2 | grep Image
    Image:        nginx:1.28
    Image:      wbitt/network-multitool
user@microk8s:~$ kubectl exec deployments/app-task2 -c 
multitool  nginx      
user@microk8s:~$ kubectl exec deployments/app-task2 -c 
multitool  nginx      
user@microk8s:~$ kubectl exec deployments/app-task2 -c multitool -it -- bash
app-task2-6f49d9b64d-5mlvt:/# curl -I app-task2:80
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Sat, 30 Nov 2024 04:04:03 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Nov 2021 14:44:02 GMT
Connection: keep-alive
ETag: "6193c3b2-264"
Accept-Ranges: bytes

```
4. Откатиться после неудачного обновления.   
Откатываюсь:
```bash
user@microk8s:~/kuber-homeworks-3.4$ kubectl rollout undo deployment app-task2 
deployment.apps/app-task2 rolled back
user@microk8s:~/kuber-homeworks-3.4$ kubectl get po 
NAME                         READY   STATUS    RESTARTS   AGE
app-task2-6f49d9b64d-2t7k8   2/2     Running   0          12s
app-task2-6f49d9b64d-5mlvt   2/2     Running   0          23m
app-task2-6f49d9b64d-kjwwg   2/2     Running   0          23m
app-task2-6f49d9b64d-l5ps2   2/2     Running   0          12s
app-task2-6f49d9b64d-rfkps   2/2     Running   0          13s
user@microk8s:~/kuber-homeworks-3.4$ kubectl describe deployments.apps app-task2 | grep Image
    Image:        nginx:1.20
    Image:      wbitt/network-multitool
```
```````````
 

 

### Задание 3*. Создать Canary deployment

1. Создать два deployment'а приложения nginx.
2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.

Написал манифесты, для создания двух деплойментов разных версий nginx, при помощи ConfigMap,сделал две разные вебстраницы.
Так же написал сервисы и ingress в котором использовал аннотации  __nginx.ingress.kubernetes.io/canary: "true"__ и __nginx.ingress.kubernetes.io/canary-weight: "50"__ для распределения траффика между версиями 50/50.   
Использовал ddns, сделал проброс на своем роутере чтобы можно было проверить из вне.   

Манифесты записал в одном файле:
```yaml

# deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
  namespace: task3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.19
          ports:
           - containerPort: 80
          volumeMounts:
            - name: config-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: config-volume
          configMap:
            name: nginx-config-v1
---
# deployment-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
  namespace: task3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
        - name: nginx
          image: nginx:1.20
          volumeMounts:
            - name: config-volume
              mountPath: /usr/share/nginx/html
      volumes:
        - name: config-volume
          configMap:
            name: nginx-config-v2
---

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config-v1
  namespace: task3

data:
  index.html: |
    <html>
    <body>
    <pre>
      V1 Nginx-1.19
      --------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||
    </pre>
    </html>
    </body>
---

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config-v2
  namespace: task3
data:
  index.html: |
    <html>
    <body>
    <pre>
      V2 Nginx-1.20
      --------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||
    </pre>
    </html>
    </body>
---

# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50" # % трафика 


spec:
  rules:
  - host: suntsovvv.tplinkdns.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: nginx-v2-service
            port:
              number: 80
---

# nginx-v1-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-v1-service
  namespace: task3
spec:
  selector:
    app: nginx
    version: v1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  externalIPs:
   - 192.168.0.105
---
# nginx-v2-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-v2-service
  namespace: task3
spec:
  selector:
    app: nginx
    version: v2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  externalIPs:
   - 192.168.0.105

---
```
Проверил результат


-----

При проверке не мог поймать смену версий , всегда попадал на одну и ту же. Но когда попробоал подключаться с разных машин, получил результат:   




Ссылка на репозиторий с манифестами https://github.com/suntsovvv/kuber-homeworks-3.4.