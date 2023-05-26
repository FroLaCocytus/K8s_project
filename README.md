<h1 align="center">Практика Docker + K8s</h1>

<h2 align="center">Работа с <code>Docker</code></h2>

Структура проекта выглядит следующим образом:
```
.
├── Dockerfile
└── hello.html
```

Создадим файл `hello.html`:
```html
<!DOCTYPE html>
<html>
<head>
  <title>Hello World</title>
</head>
<body>
  <h1>Hello, World!</h1>
</body>
</html>
```

Создадим файл `Dockerfile`:

```Dockerfile
# Базовый image
FROM python:3.10-alpine

# Переменные, используемые для создания окружения, в котором запустится приложение
ARG USER=app 
ARG UID=1001
ARG GID=1001

# Создание пользователя операционной системы и его домашнего каталога
RUN addgroup -g ${GID} -S ${USER} \
   && adduser -u ${UID} -S ${USER} -G ${USER} \
   && mkdir -p /app \
   && chown -R ${USER}:${USER} /app
USER ${USER}

# Переход в каталог /app
WORKDIR /app

# Копирование файла hello.html в домашний каталог
COPY --chown=$USER:$USER hello.html /app

# Команда запуска web-сервера
CMD ["python", "-m", "http.server", "8000"]
```

Запустим `демон` если он не запущен:
```bash
sudo service docker start
```

Авторизуемся в `Docker registry`. В качестве пароля введем `Access Token`:
```shell
docker login -u frolacocytus
```

Соберём `Docker image`:
```bash
docker build -t frolacocytus/project:1.0.0 --network host .
```
Проверим наличие нашего `Docker image`:
```bash
docker image ls
```

Запустим `Docker container` и проверем работоспособность web-приложения (будет доступно по адресу http://127.0.0.1:8000/hello.html):
```bash
docker run -ti --rm -p 8000:8000 --name project --network host frolacocytus/project:1.0.0
```

Выложим наш `Docker image` на `Docker Hub`:
```bash
docker push frolacocytus/project:1.0.0
```
---
<h2 align="center">Работа с <code>Kubernetes</code></h2>

Запустим кластер `Kubernetes`. Для удобства переопределим путь к `kubeconfig`:

```bash
export KUBECONFIG=$HOME/.kube/minikube
```
```bash
minikube start --embed-certs
```
Посмотрим статус запущенного кластера:
```bash
minikube status
```

Создадим `Deployment manifest` (количество реплик равно 2). Для этого создадим файл `deployment_project.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: project
spec:
  replicas: 2
  selector:
    matchLabels:
      app: project
  template:
    metadata:
      labels:
        app: project
    spec:
      containers:
      - name: docker-project
        image: frolacocytus/project:1.0.0
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /hello.html
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /hello.html
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
```

Применим `manifest`:
```bash
kubectl apply --filename deployment_project.yaml --namespace default
```

Посмотрим, подробную информацию о `Deployment` (ждём пока `Probes` пройдут успешно):
```bash
kubectl describe deployment web --namespace default
```

Посмотрим на количество `Pod` в `Deployment` (проверим что их два) и убедимся что они готовы к работе:
```bash
kubectl get deployments --namespace default
```

Пробросим наружу порт нашего проекта и проверем работоспособность web-приложения (будет доступно по адресу http://127.0.0.1:8080/hello.html):
```bash
kubectl port-forward --address 0.0.0.0 deployment/web 8080:8000
```
---

<h2 align="center">Результат команды <code>kubectl describe deployment web</code></h2>

```
Name:                   web
Namespace:              default
CreationTimestamp:      Sat, 27 May 2023 02:09:33 +0300
Labels:                 app=project
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=project
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=project
  Containers:
   docker-project:
    Image:        frolacocytus/project:1.0.0
    Port:         8000/TCP
    Host Port:    0/TCP
    Liveness:     http-get http://:8000/hello.html delay=10s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get http://:8000/hello.html delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-766d9c544 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  18m   deployment-controller  Scaled up replica set web-766d9c544 to 2
```