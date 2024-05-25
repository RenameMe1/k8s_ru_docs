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
### Альтернативы Secrets

Прежде чем использовать [[Secret]] для защиты конфиденциальной информации, вы можете выбрать из альтернатив.

Здесь некоторые ваши варианты:
- Если ваши облачные компоненты в аутентификации в другом приложении и вы знаете, что оно запущено в пределах этого же Kubernetes кластера, вы можете использовать [[Authenticating#Service account tokens|ServiceAccount]] и его токен для идентификации вашего клиента.
- Есть сторонние инструменты, которые вы можете запустить в пределах или снаружи вашего кластера, которые будут управлять чувствительными данными. Для примера, служба к которой [[Pod]] получают доступ через HTTPS, которая раскрывает [[Secret]] если клиент корректно прошел аутентификацию (для примера, с [[Authenticating#Service account tokens|ServiceAccount]])
- Для аутентификации, вы можете реализовать собственный подписчик для сертификатов X.509 и использовать [[Certificates and Certificate Signing Requests|CertificateSigningRequests]], что бы позволить собственному подписчику выдавать сертификаты для [[Pod]] который в них нуждаются
- Вы можете использовать [[Device Plugins]] для предоставления аппаратного обеспечения для шифрования на локальной [[Nodes]] для указанных [[Pod]]. Для примера, мы можете запланировать доверенные [[Pod]] на узлах, которые предоставляют  Trusted Platform Module, настроенный вне группы.

Вы можете также объединять два или более этих варианта, включая варианты использования сами объекты [[Secret]].

Для примера: реализовав (или развернув) [[Operator pattern|operator]] (специальный контроллер используемый для управления пользовательскими ресурсами) извлекающий короткоживущие токены сессии из внешних сервисов и затем создавая [[Secret]] основанный на этих короткоживущих токенах сессии. [[Pod]] работающие в вашем кластере могут использовать токены сессии, а  [[Operator pattern|operator]] гарантирует, что они верны. Это разделение гарантирует, что вы можете запустить [[Pod]], которые не подозревают о точных механизмах для издания и обновления этих токенов сессии.
## Типы Secret

При создании [[Secret]], вы можете указать его тип используя поле `type` ресурса [[Secret]] или определенный аналог флага команды `kubectl` в терминале (если доступно). Тип [[Secret]] используется для облегчения программной обработки данных [[Secret]].

Kubernetes предоставляет несколько встроенных типов для таких общих сценариев использования. Эти типы различаются в условиях выполнения проверок (валидации) и ограничены Kubernetes. 

| Built-in Type                         | Usage                                          |
| ------------------------------------- | ---------------------------------------------- |
| `Opaque`                              | произвольные определенные пользователем данные |
| `kubernetes.io/service-account-token` | токен ServiceAccount                           |
| `kubernetes.io/dockercfg`             | сериализация `~/.dockercfg` файла              |
| `kubernetes.io/dockerconfigjson`      | сериализация `~/.docker/config.json` файла     |
| `kubernetes.io/basic-auth`            | учетные данные для базовой аутентификации      |
| `kubernetes.io/ssh-auth`              | учетные данные для SSH аутентификации          |
| `kubernetes.io/tls`                   | данные для TLS клиента или сервера             |
| `bootstrap.kubernetes.io/token`       | данные токена начальное загрузки               |

Вы можете определить и использовать ваш собственный тип [[Secret]] назначив не пустую строку как значение `type` для объекта [[Secret]] (пустая строка будет обработана как тип `Opaque`). 

Kubernetes не навязывает какие-либо ограничения на имя типа. Однако, если вы используете один из встроенных типов, вы должны соответствовать всем требованиям, определенным для этого типа.

Если вы определяете тип [[Secret]], который будет доступен для публичного использования, следуйте соглашению и структурируйте тип [[Secret]] таким образом, что бы перед именем типа было ваше доменное имея , разделенное `/`. Для примера: `cloud-hosting-example.net/cloud-api-credentioals`.
### Opaque Secrets

`Opaque` стандартный тип [[Secret]] если вы явно не указали тип в [[Secret]] манифесте. Когда вы создаете [[Secret]] используя `kubectl`, вы должны использовать подкоманду `generic` для  обозначения `Opaque` типа. Для примера, следующая команда создает пустой [[Secret]] типа `Opaque`:

```shell
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

Вывод выглядит как:

``` shell
NAME           TYPE     DATA   AGE
empty-secret   Opaque   0      2m6s
```

Колонка `DATA` показывает количество элементов данных в [[Secret]]. В этом случае, `0` означает, что вы создали пустой [[Secret]].
### ServiceAccount токен Secrets

Тип [[Secret]] `kubernetes.io/service-account-token` используется для хранения токена учетных данных идентифицирующий [[Authenticating#Service account tokens|ServiceAccount]]. Это устаревший механизм, который предоставляет долго живущие учетные данные [[Authenticating#Service account tokens|ServiceAccount]] для [[Pod]].

В Kubernetes v1.22 и позднее, рекомендуемый подход - получение короткоживущих, автоматически сменяющийся токен [[Authenticating#Service account tokens|ServiceAccount]] используемый вместо [[TokenRequest]]. Вы можете получить эти короткоживущие токены использующие следующие методы:
- Вызывает API `TokenRequest` напрямую или используя API клиент например как `kubectl`. Для примера: вы можете использовать команду [`kubectl create token`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-token-em-).
- Запросите смонтированный токен в [[Managing Service Accounts#Bound service account token volume mechanism|projected volume]] вашего [[Pod]] манифеста. Kubernetes создает токен и монтирует его в [[Pod]]. Токен автоматически признается не действительным когда [[Pod]] в который он смонтирован удален. Подробности смотри в [[Configure Service Accounts for Pods#Launch a Pod using service account token projection|Запуск Pod используя  service account token проекцию токена сервисного аккаунта]] 

> [!NOTE]
> Вы должны создать ServiceAccount токен Secret, только если вы не можете использовать API `TokenRequest` для получения токена и угрозы безопасности, связанной с сохранением учетных данных токена, срок действия которых не истекает в API объекте доступном вам для чтения.

Когда вы используете тип [[Secret]], вам необходимо убедиться что аннотация `kubernetes.io/service-account.name` установлена на существующее имя [[Authenticating#Service account tokens|ServiceAccount]]. Если вы создаете и [[Authenticating#Service account tokens|ServiceAccount]] и [[Secret]] объекты, вы должны создать первым объектом [[Authenticating#Service account tokens|ServiceAccount]]. 

После создания [[Secret]], Kubernetes [[Controllers|контроллер]] заполнит некоторые другие поля аннотацией `kubernetes.io/service-account.uid` и `token` ключ в поле `data`, который заполняется токеном аутентификации.

Для следующем примере конфигурация заявляет [[Authenticating#Service account tokens|ServiceAccount]] токен в [[Secret]]

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  extra: YmFyCg==
```

После создания [[Secret]], дождитесь, пока Kubernetes введет ключ `token` в поле `data`.

Смотрите документацию [[Service Account]] для большей информации о том, как работает [[Service Account]]. Вы так же можете проверить [[Pod]] поля `automountServiceAccountToken` и `serviceAccountName` для информации о том, как ссылаться на учетные данные [[Service Account]] из модулей. 
### Docker config Secrets

Если вы создаете [[Secret]] для хранения учетных данных для доступа к реестру образов контейнеров, вы должны использовать один из следующих значений `type` для этого [[Secret]].
- `kubernetes.io/dockercfg`:  сохраните стерилизованный файл  `~/.dockercfg`, который является устаревшим форматом для настройки командной строки Docker. [[Secret]] поле `data` содержит ключ `.dockercfg` значение которого это контент файла `~/.dockercfg` в закодированном base64 формата.
- `kubernetes.io/dockerconfigjson`: сохраните стерилизованный JSON, которые следует таким же правилам форматирования как и файл `~/.docker/config.json`, который является новым форматом для `~/.dockercfg`. [[Secret]] поле `data` должно содержать ключ `.dockerconfigjson`, для которого значение является контентом закодированного формата base64  `~/.docker/config.json` файла.

Ниже пример для [[Secret]] типа `kubernetes.io/dockercfg`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  .dockercfg: |
    eyJhdXRocyI6eyJodHRwczovL2V4YW1wbGUvdjEvIjp7ImF1dGgiOiJvcGVuc2VzYW1lIn19fQo= 
```

> [!NOTE]
> Если вы не хотите выполнять base64 кодировку, вы можете выбрать для использование поле `stringData` вместо `data`.

Когда вы создаете Docker конфигурацию используя манифест [[Secret]] , API сервер проверяет будь то ожидаемый существующий ключ поля `data` и проверяет его, если предоставляемое  значение может быть разобрано как корректный JSON. Сервер API не проверяет, действительно ли JSON является файлом конфигурации Docker.

Вы так же можете использовать `kubectl` для создания [[Secret]] доступа к реестру контейнера,  когда у вас нет файла конфигурации Docker:

```shell
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

Команда создает [[Secret]] `kubernetes.io/dockerconfigjson` типа.

Для извлечения поля `.data.dockerconfigjson` из этого нового [[Secret]] и декодирования данных:

``` shell
kubectl get secret secret-tiger-docker -o jsonpath='{.data.*}' | base64 -d
```

Вывод команды аналогичен следующему JSON документу (который так же является допустимым конфигурационным файлом Docker)

```json
{
  "auths": {
    "my-registry.example:5000": {
      "username": "tiger",
      "password": "pass1234",
      "email": "tiger@acme.example",
      "auth": "dGlnZXI6cGFzczEyMzQ="
    }
  }
}
```

> [!CAUTION]
> Значение `auth` закодировано в base64; это скрытно, но не секретно. Кто угодно может прочитать этот [[Secret]] может узнать токен владельца доступа к реестру.
> 
> Рекомендуется использование [[Configure a kubelet image credential provider|поставщиков учетных данных]] для динамического и надежного получения конфиденциальных данных по требованию.
 
### Basic authentication Secret

Тип `kubernetes.io/basic-auth` предоставлен для хранения учетных данных необходимых для базовой аутентификации. При использовании этого типа [[Secret]], поле `data` может должен содержать один из следующих двух ключей:
- `username`: Имя пользователя для аутентификации
- `password`: Пароль или токен для аутентификации

Оба значения для выше указанных ключей закодированные base64 строки. Вы можете альтернативно предоставить не кодированные данные используя `stringData` в манифесте [[Secret]].

Следующий манифест это пример [[Secret]] для базовой аутентификации:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin # Обязательное поле для kubernetes.io/basic-auth
  password: t0p-Secret # Обязательное поле для kubernetes.io/basic-auth
```

> [!NOTE]
> Поле `strigData` для [[Secret]] не работает хорошо с применением на стороне сервера

Базовый аутентификационный тип [[Secret]] предоставлен только для удобства. Вы можете создать [[#Opaque Secrets]] для учетных данных используемых в базовой аутентификации. Однако, использование заданного и публичного [[Secret]] типа (`kubernetes.io/basic-auth`) помогает другим людям понимать цель вашего [[Secret]] и устанавливает соглашение какие имена ключей ожидать. Kubernetes API проверяет какие обязательные ключи указаны для этого типа [[Secret]].
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


