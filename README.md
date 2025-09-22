# Домашнее задание к занятию «Настройка приложений и управление доступом в Kubernetes»

> Репозиторий: hw-39\
> Выполнил: Асадбек Асадбеков\
> Дата: сентябрь 2025

## Задание 1: Работа с ConfigMaps

#### Задача
Развернуть приложение (nginx + multitool), решить проблему конфигурации через ConfigMap и подключить веб-страницу.

### Манифесты

**configmap-web.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Страница из ConfigMap</title>
    </head>
    <body>
      <h1>Привет от Kubernetes!</h1>
      <p>Это страница, настроенная через ConfigMap</p>
    </body>
    </html>
```
</details>

**configmap-nginx.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
```
</details>

**deployment.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: web-content
          mountPath: /usr/share/nginx/html
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 80
        command: ["sh", "-c", "sleep infinity"]
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: web-content
        configMap:
          name: web-content
```
</details>

**service.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
</details>

### Скриншоты

![Создание ConfigMap](https://github.com/asad-bekov/hw-39/blob/main/img/1.PNG)

![Создание Service](https://github.com/asad-bekov/hw-39/blob/main/img/2.PNG)

![Проверка доступности через curl](https://github.com/asad-bekov/hw-39/blob/main/img/3.PNG)

---

## Задание 2: Настройка HTTPS с Secrets

#### Задача
Развернуть приложение с доступом по HTTPS, используя самоподписанный сертификат.

### Генерация сертификатов

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"
```

### Манифесты

**secret-tls.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-key>
```
</details>

**ingress-tls.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```
</details>

### Скриншоты

![Генерация SSL-сертификата](https://github.com/asad-bekov/hw-39/blob/main/img/4.PNG)

![Создание TLS Secret](https://github.com/asad-bekov/hw-39/blob/main/img/5.PNG)

![HTTPS доступ через curl -k](https://github.com/asad-bekov/hw-39/blob/main/img/6.PNG)

---

## Задание 3: Настройка RBAC

#### Задача
Создать пользователя с ограниченными правами (только просмотр логов и описания подов).

### Генерация сертификатов пользователя

```bash
# Генерация приватного ключа
openssl genrsa -out developer.key 2048

# Создание CSR
openssl req -new -key developer.key -out developer.csr -subj "/CN=developer"

# Подписание сертификата CA кластера
sudo openssl x509 -req -in developer.csr -CA /var/snap/microk8s/8355/certs/ca.crt -CAkey /var/snap/microk8s/8355/certs/ca.key -CAcreateserial -out developer.crt -days 365
```

### Манифесты

**role-pod-reader.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer
  namespace: default
rules:
- apiGroups: [""]
  resources:
    - pods
    - pods/log
  verbs:
    - get
    - list
    - watch
    - describe
```
</details>

**rolebinding-developer.yaml:**

<details>
<summary>Показать</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```
</details>

### Скриншот

![Доступ к pods и запрет доступа к services](https://github.com/asad-bekov/hw-39/blob/main/img/7.PNG)

