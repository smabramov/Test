# Задание

1. Что такое CI/CD?

### Ответ

CI/CD (Continuous Integration / Continuous Delivery или Continuous Deployment) — это набор практик и инструментов для автоматизации процессов разработки, тестирования и развертывания программного обеспечения.

2. Напишите простой демон для systemd, который будет поддерживать работу процесса и перезапускаться в случае выхода из строя процесса.

### Ответ

Создадим промстой скрипт который будет работат, как демон:

```
#!/bin/bash

# Простой демон, который просто пишет в лог раз в секунду
while true; do
    echo "$(date): Демон работает..." >> /var/log/simple_daemon.log
    sleep 1
done
```
Сделаем его исполняемым:

```
sudo chmod +x /usr/local/bin/simple_daemon.sh
```

Создаем файл /etc/systemd/system/simple_daemon.service:

```
[Unit]
Description=Простой демон с автоматическим перезапуском
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash /usr/local/bin/simple_daemon.sh
Restart=always
RestartSec=5
# Рабочая директория (обязательно укажите!)
WorkingDirectory=/tmp
# Перенаправление логов
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```
Запускаем демон:

```
sudo systemctl daemon-reload                  # Перечитываем конфигурацию systemd
sudo systemctl enable simple_daemon.service   # Включаем автозапуск
sudo systemctl start simple_daemon.service    # Запускаем демон
sudo systemctl status simple_daemon.service   # Проверяем статус

```
Вывод:

```
serg@debian:~/git/Test$ sudo systemctl status simple_daemon.service
● simple_daemon.service - Простой демон с автоматическим перезапуском
     Loaded: loaded (/etc/systemd/system/simple_daemon.service; enabled; preset: enabled)
     Active: active (running) since Mon 2025-04-21 10:41:19 MSK; 12min ago
   Main PID: 9614 (bash)
      Tasks: 2 (limit: 9461)
     Memory: 620.0K
        CPU: 2.325s
     CGroup: /system.slice/simple_daemon.service
             ├─ 9614 /bin/bash /usr/local/bin/simple_daemon.sh
             └─12570 sleep 1
```

Проверяем логи:

```
tail -f /var/log/simple_daemon.log
```

```
Mon Apr 21 10:48:44 MSK 2025: Демон работает...
Mon Apr 21 10:48:45 MSK 2025: Демон работает...
Mon Apr 21 10:48:46 MSK 2025: Демон работает...
```
Если убить процесс вручную, то демон перезапустится через 5 сек из-за Restart=always.

```
sudo kill 9614
```

3. Что такое inode в Linux?

### Ответ

Inode (index node) — это структура данных в файловых системах Linux/Unix, которая хранит метаданные о файле (всё, кроме имени и содержимого).

Что хранит inode:

Тип файла (обычный файл, директория, символьная ссылка и т.д.).

Права доступа (rwx для владельца, группы и других).

UID/GID (владелец и группа).

Размер файла в байтах.

Временные метки (дата создания, изменения, доступа).

Ссылки на блоки данных на диске (где физически хранится содержимое).

Количество жёстких ссылок на файл.


Можно сказать, что inode-это паспорт файла, а имя файла - это просто кличка.

4. Сделайте реализацию blue/green стратегии деплоймента для Kubernetes на основе деплойментов, сервиса и ingress’а. Опишите как переключать версии.

### Ответ

Файл deployment-blue.yaml:

```
app: my-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app
        image: my-app:v1
        ports:
        - containerPort: 80
```
Файл deployment-green.yaml:

```
metadata:
  name: app-green
  labels:
    version: green
spec:
  template:
    metadata:
      labels:
        version: green
    spec:
      containers:
      - image: my-app:v2
```

Применяем:

```
kubectl apply -f deployment-blue.yaml
kubectl apply -f deployment-green.yaml
```
Файл service.yaml:

```
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: my-app
    version: blue  # Начинаем с blue
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Применяем:

```
kubectl apply -f service.yaml
```
Файл ingress.yaml:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```
Применяем:

```
kubectl apply -f ingress.yaml
```
Чтобы переключить трафик на Green:

1. Обновляем селектор Service:

```
kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'
```

2. Проверяем переключение:

```
kubectl get pods -l version=green
curl my-app.example.com
```
Если Green работает некорректно, можно откатиться на Blue:

```
kubectl patch service app-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

Автоматизация. Можно использовать helm.


```
# Пример с Helm
helm upgrade my-app --set deployment.version=green
```
Проверка:

```
kubectl get deployments,svc,pods,ingress
```

5. Напишите политику для AWS S3 бакета, которая разрешает доступ только с определенных IP адресов.

Пример политики (подставьте свои значения):

```
{
    "Version": "2012-10-17",
    "Id": "S3PolicyRestrictByIP",
    "Statement": [
        {
            "Sid": "AllowAccessFromTrustedIPs",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::YOUR_BUCKET_NAME",
                "arn:aws:s3:::YOUR_BUCKET_NAME/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": [
                        "192.0.2.0/24",   // Пример: разрешенная подсеть
                        "203.0.113.42/32" // Пример: конкретный IP
                    ]
                }
            }
        },
        {
            "Sid": "DenyAllOtherIPs",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::YOUR_BUCKET_NAME",
                "arn:aws:s3:::YOUR_BUCKET_NAME/*"
            ],
            "Condition": {
                "NotIpAddress": {
                    "aws:SourceIp": [
                        "192.0.2.0/24",
                        "203.0.113.42/32"
                    ]
                }
            }
        }
    ]
}
```
Как применить политику?

Замените в коде:

YOUR_BUCKET_NAME → имя вашего бакета.

192.0.2.0/24, 203.0.113.42/32 → ваши доверенные IP/подсети.

Добавьте политику в бакет:

Откройте AWS S3 Console → выберите бакет → вкладка Permissions → Bucket Policy.

Вставьте JSON и сохраните.



Дополнительные настройки.

Ограничение для конкретных операций (например, только GetObject):
Замените "s3:*" на "s3:GetObject".
Доступ для IAM-ролей/пользователей (если нужно обойти IP-ограничение):
Добавьте правило с "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID:user/NAME" }.

Важно!

Если у бакета включен публичный доступ, S3 игнорирует эту политику.
Отключите в настройках бакета:
Permissions → Block Public Access → Enable all settings.

6. Объясните паттерны IaaS/PaaS/SaaS на примере пиццы.

1. IaaS (Infrastructure as a Service) — "Собери сам"

Вам привозят сырую пиццу (тесто, соус, сыр, начинки), но печь и нарезать нужно самому.

Что делаем:

Настраиваете сервер (как печь пиццу).
Устанавливаете ОС, ПО (как выбирать температуру и время).
Управляете безопасностью (как резать и подавать).

Что предоставляет провайдер:

Виртуальные машины (EC2), сети, хранилища (S3).

2. PaaS (Platform as a Service) — "Приготовь из готовых ингредиентов"

Вам привозят почти готовую пиццу, но вы можете добавить свои топпинги.

Что делаем:

Разрабатываете приложение (выбираете начинку).
Не думаете о серверах, ОС и сетях.

Что предоставляет провайдер:

Готовую платформу для запуска кода (Heroku, App Engine).

Готовую платформу для запуска кода

3. SaaS (Software as a Service) — "Просто ешь"

Вы приходите в пиццерию и едите готовую пиццу.

Что делаем:

Только пользуетесь (никакого управления серверами).

Что предоставляет провайдер:

Полностью готовый софт (Gmail, Slack).

Вывод

    IaaS — контроль на вашей стороне (как сырая пицца).

    PaaS — вы сосредоточены на разработке (как выбор топпингов).

    SaaS — вам нужен только результат (как готовая пицца).


💡 Как запомнить?

    IaaS = "Infrastructure" (серверы, сети) → "Собери сам".

    PaaS = "Platform" (готовый стек для кода) → "Допили начинку".

    SaaS = "Software" (весь софт готов) → "Просто ешь".



7.     Есть условное Node.js приложение и неправильно написанный Dockerfile, который не будет кэшироваться и будет занимать много места. Нужно переписать его в соответствии с best-practice

#плохой файл
FROM ubuntu:18.04
COPY ./src /app
RUN apt-get update -y
RUN apt-get install -y nodejs
RUN npm install
ENTRYPOINT [“npm”]
CMD [“run”, “prod”]

### Ответ 

```
# Используем легкий официальный образ Node.js на Alpine Linux
FROM node:18-alpine

# Создаем рабочую директорию
WORKDIR /app

# Копируем package.json и package-lock.json (или yarn.lock)
COPY package*.json ./

# Устанавливаем зависимости (кэшируется, если package.json не менялся)
RUN npm install --production

# Копируем остальные файлы проекта
COPY ./src ./

# Указываем переменные среды (опционально)
ENV NODE_ENV=production

# Запускаем приложение
CMD ["npm", "run", "prod"]
```

8. С помощью чего можно ограничить в Kubernetes сетевое взаимодействие между подами? Приведите пример. Надо ли отдельно включать данный механизм?

### Ответ

Для управления сетевым трафиком между подами в Kubernetes используется NetworkPolicy. Этот механизм позволяет задавать правила, определяющие, какие пода могут общаться друг с другом, а какие — нет.


NetworkPolicy работает только при наличии сетевого плагина (CNI), который его поддерживает. Например:

Calico (наиболее популярный, поддерживает сложные политики)

Cilium (на базе eBPF, с улучшенной производительностью)

Weave Net (поддерживает базовые NetworkPolicy)

kube-router

Проверить, поддерживает ли ваш кластер NetworkPolicy:

```
kubectl get networkpolicies --all-namespaces
```
Если список пуст, но ошибок нет — значит, NetworkPolicy поддерживается, но политик пока нет.

9. Что такое POSIX?


POSIX (Portable Operating System Interface) — это набор стандартов, определяющих совместимость между Unix-подобными операционными системами. Его цель — гарантировать, что программы, написанные для одной Unix-системы, будут работать и на других без изменений.

10. Приведите основные типы DNS записей и для чего они используются?

### Ответ.

1. A-запись (Address Record)

    Назначение: Связывает доменное имя с IPv4-адресом.

    Пример:
    example.com. A 192.0.2.1
    Это означает, что домен example.com указывает на сервер с IP 192.0.2.1.

2. AAAA-запись (IPv6 Address Record)

    Назначение: Аналогична A-записи, но для IPv6-адресов.

    Пример:
    example.com. AAAA 2001:db8::1
    Домен example.com связан с IPv6-адресом 2001:db8::1.

3. CNAME-запись (Canonical Name)

    Назначение: Создает псевдоним (алиас) для домена, перенаправляя его на другое доменное имя.
    Нельзя использовать для корневого домена (например, example.com → ошибка).

    Пример:
    www.example.com. CNAME example.com.
    Запрос к www.example.com перенаправляется на example.com.

4. MX-запись (Mail Exchange)

    Назначение: Указывает почтовые серверы, принимающие email для домена.
    Чем меньше значение приоритета, тем выше приоритет сервера.

    Пример:
    example.com. MX 10 mail1.example.com.
    example.com. MX 20 mail2.example.com.
    Почта для example.com сначала отправляется на mail1.example.com, затем на mail2.example.com.

5. TXT-запись (Text Record)

    Назначение: Хранение произвольного текста. Используется для:

        SPF (проверка отправителя email).

        DKIM (цифровая подпись писем).

        DMARC (политика обработки почты).

        Верификации владения доменом (например, для Google Search Console).

    Пример SPF:
    example.com. TXT "v=spf1 include:_spf.google.com ~all"

6. NS-запись (Name Server)

    Назначение: Указывает DNS-серверы, отвечающие за доменную зону.

    Пример:
    example.com. NS ns1.example.com.
    example.com. NS ns2.example.com.
    DNS-запросы для example.com обрабатываются серверами ns1.example.com и ns2.example.com.

7. SOA-запись (Start of Authority)

    Назначение: Служебная запись с метаданными о доменной зоне:

        Первичный DNS-сервер.

        Email администратора.

        Тайминги (обновление, кэширование).

    Пример:
    plaintext

    example.com.    SOA    ns1.example.com. admin.example.com. (
                    2024022001 ; Serial number
                    3600       ; Refresh (1 hour)
                    1800       ; Retry (30 mins)
                    604800     ; Expire (1 week)
                    86400 )    ; TTL (1 day)

8. PTR-запись (Pointer Record)

    Назначение: Обратный DNS — связывает IP-адрес с доменным именем.
    Используется для антиспама и логов.

    Пример:
    1.2.0.192.in-addr.arpa. PTR example.com.
    IP 192.0.2.1 соответствует домену example.com.

9. SRV-запись (Service Record)

    Назначение: Указывает серверы для специфических служб (VoIP, LDAP, XMPP).
    Формат: _сервис._протокол.домен.

    Пример для SIP (VoIP):
    _sip._tcp.example.com. SRV 10 60 5060 sipserver.example.com.
    Где:

        10 — приоритет.

        60 — вес.

        5060 — порт сервера.

        sipserver.example.com. — целевой сервер.

10. CAA-запись (Certification Authority Authorization)

    Назначение: Определяет, какие центры сертификации (CA) могут выпускать SSL-сертификаты для домена.
    Защищает от несанкционированного выпуска сертификатов.

    Пример:
    example.com. CAA 0 issue "letsencrypt.org"
    Только Let’s Encrypt может выпускать сертификаты для example.com.