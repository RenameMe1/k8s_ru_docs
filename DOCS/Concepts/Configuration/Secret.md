---
tags:
  - k8s
---
# Secrets

[[Secret]] это объект содержащий малое количество конфиденциальных данных таких как пароль, токен или ключ. В противном случае такая информация могла быть помещена в спецификацию [[Pod]]'а или образ контейнера. Использование Secret означает что вам не нужно включать конфиденциальные данные в ваш код приложения.

Потому что [[Secret]] могут быть созданы независимо от [[Pod]] использующего их, есть маленький риск, что [[Secret]] (и его данные) будут не защищены в течении рабочего процесса создания, просмотра и редактирования [[Pod]]. Kubernetes и приложения, которые работают в вашем кластере, могут также добавлять дополнительные меры предосторожности с [[Secret]], такие как избегание записи конфиденциальных данных в энергонезависимом хранилище. 

[[Secret]] аналогичны [[ConfigMap]], но специально предназначены для хранения конфиденциальных данных. 

> [!CAUTION]
> ##### Осторожно
>Kubernetes [[Secret]] по умолчанию хранящиеся не зашифровано в базовом хранилище данных (etcd) API сервера. Кто угодно с доступом к API может получить или модифицировать [[Secret]] и также может любой с доступом к etcd. Дополнительно, любой кто авторизован для создания [[Pod]] в [[namespase]] может использовать этот доступ для чтения любого [[Secret]] в этом [[namespase]]; это включает косвенный доступ такой как возможность создания [[Deployment]] 
>
>Что бы  безопасно использовать [[Secret]] выполните как минимум следующие шаги:
>1. [[Encrypting Confidential Data at Rest|Включите шифрование при хранении]]
>2. [[Authorization|Включите или настройте RBAC правила]] с наименьшими привилегиями доступа для [[Secret]]
>3. Ограничьте доступ к [[Sercet]] для определенных контейнеров
>4. [Рассмотрите использование поставщиков внешних Secret хранилищ](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver)
>
>Дополнительные рекомендации по управлению и повышению безопасности ваших Secret'ов, обратитесь [[Good practices for Kubernetes Secrets]]

Смотрите [[Secret#Information security for Secrets|информационная безопасность для Secret]] для больших деталей.

## Использования Secrets

Вы можете использовать [[Secret]] для следования следующим целям:
- [[Distribute Credentials Securely Using Secrets#Define container environment variables using Secret data|Установка переменных окружения для контейнера]]
- [[Distribute Credentials Securely Using Secrets#Example Provide prod/test credentials to Pods using Secrets|Предоставление учетных данных, таких как SSH ключи или пароли в Pod]]
- [[Pull an Image from a Private Registry|Позволить kubelet получать образы контейнеров из приватных  реестров]]
Управляющий уровень Kubernetes также использует [[Secret]]; для примера, [[#Bootstrap token Secrets|Секреты токенов начальной загрузки]] это механизм помогающий автоматизировать регистрацию [[Nodes]]

### Пример использования: dotfiles в Secret томе

Вы можете создать ваши данные "скрытыми" определив ключ, который начинается с точки. Этот ключ представляет собой dotfile или "скрытый" файл. Для примера, когда следующий [[Secret]] монтируется в том `secret-volume`, том будет содержать одиночный файл, с названием .secret-file и `dotfile-test-container` будет иметь этот файл представленный по пути `/etc/secret-volume/.secret-file`. 

> [!NOTE]
>  Файл начинающийся с символа точки скрыт для вывода `ls -l`, вы должны использовать `ls -la`, что бы увидеть его, при просмотре контента репозитория.

```  yaml
# secret/dotfile-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
  containers:
    - name: dotfile-test-container
      image: registry.k8s.io/busybox
      command:
        - ls
        - "-l"
        - "/etc/secret-volume"
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
```


### Пример использования: Secret видимый одному контейнеру в Pod

Рассмотрим программу, которой необходимо оперировать HTTP запросами, делая некоторую сложную бизнес логику, и затем подписывать некоторые сообщения с помощью HMAC. Поскольку он имеет сложную логику приложения, на сервере может быть незамеченный эксплойт для удаленного чтения файлов, который может предоставить злоумышленнику доступ к закрытому ключу.

Это может быть разделенный на два процесса в двух контейнерах: frontend контейнер, который обрабатывает пользовательское взаимодействие и бизнес логику, но который не имеет доступа к приватному ключу; и контейнер для подписи, который имеет доступ к приватному ключу и отвечает запросам простой подписи из frontend (для примера, через сеть localhost).

С таким разделенным подходом, теперь злоумышленнику приходится обманом заставить сервер приложений выполнить что-то довольно произвольное, что может оказаться сложнее, чем заставить его прочитать файл.
### Alternatives to Secrets
## Types of Secret
### Opaque Secrets
### ServiceAccount token Secrets
### Docker config Secrets
### Basic authentication Secret
### SSH authentication Secrets
### TLS Secrets
### Bootstrap token Secrets
## Working with Secrets
### Creating a Secret
#### Constraints on Secret names and data
#### Size limit
### Editing a Secret
### Using a Secret
#### Optional Secrets
### Using Secrets as files from a Pod
### Using Secrets as environment variables
### Container image pull Secrets
#### Using imagePullSecrets
### Using Secrets with static Pods
## Immutable Secrets
### Marking a Secret as immutable
## Information security for Secrets
### Configure least-privilege access to Secrets
## What's next


