---
tags:
  - k8s
---
# Secrets

Secret это объект содержащий малое количество конфиденциальных данных таких как пароль, токен или ключ. В противном случае такая информация могла быть помещена в спецификацию [[Pod]]'а или образ контейнера. Использование Secret означает что вам не нужно включать конфиденциальные данные в ваш код приложения.

Потому что Secret объект может быть созданы независимо от [[Pod]] использующего их, есть маленький риск, что Secret объект (и его данные) будут не защищены в течении рабочего процесса создания, просмотра и редактирования [[Pod]]. Kubernetes и приложения, которые работают в вашем кластере, могут также добавлять дополнительные меры предосторожности с Secret объектом, такие как избегание записи конфиденциальных данных в энергонезависимом хранилище. 

Secret объекты аналогичны [[ConfigMap]] объектам, но специально предназначены для хранения конфиденциальных данных. 

> [!CAUTION]
> ##### Осторожно
>Kubernetes Secret объекты по умолчанию хранятся не зашифровано в базовом хранилище данных (etcd) API сервера. Кто угодно с доступом к API может получить или модифицировать Secret объект и также это может любой с доступом к etcd. Дополнительно, любой кто авторизован для создания [[Pod]] в [[namespase]] может использовать этот доступ для чтения любого Secret объекта в этом [[namespase]]; это включает косвенный доступ такой как возможность создания [[Deployment]] 
>
>Что бы  безопасно использовать Secret объекты выполните как минимум следующие шаги:
>1. [[Encrypting Confidential Data at Rest|Включите шифрование при хранении]]
>2. [[Authorization|Включите или настройте RBAC правила]] с наименьшими привилегиями доступа для Secret объектов
>3. Ограничьте доступ к Sercet объектам для определенных контейнеров
>4. [Рассмотрите использование поставщиков внешних хранилищ конфиденциальных данных](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver)
>
>Дополнительные рекомендации по управлению и повышению безопасности ваших Secret объектов, смотрите в [[Good practices for Kubernetes Secrets]]

Смотрите [[Secret#Information security for Secrets|информационная безопасность для Secret]] для более подробной информации.

## Использования Secrets объекта

Вы можете использовать Secret объекты для преследования следующих целей:
- [[Distribute Credentials Securely Using Secrets#Define container environment variables using Secret data|Установка переменных окружения для контейнера]]
- [[Distribute Credentials Securely Using Secrets#Example Provide prod/test credentials to Pods using Secrets|Предоставление учетных данных, таких как SSH ключи или пароли в Pod]]
- [[Pull an Image from a Private Registry|Позволить kubelet получать образы контейнеров из приватных  реестров]]
Управляющий уровень Kubernetes также использует Secret объекты; для примера, [[#Bootstrap token Secrets|Secret объект токенов начальной загрузки]] это механизм помогающий автоматизировать регистрацию [[Nodes]]

### Пример использования: dotfiles в томе Secret объекта

Вы можете создать ваши данные "скрытыми" определив ключ, который начинается с точки. Этот ключ представляет собой dotfile или "скрытый" файл. Для примера, когда следующий Secret объект монтируется в том `secret-volume`, том будет содержать одиночный файл, с названием `.secret-file` и `dotfile-test-container` будет содержать этот файл представленный по пути `/etc/secret-volume/.secret-file`. 

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


### Пример использования: Secret объект видимый одному контейнеру в Pod

Рассмотрим программу, которой необходимо оперировать HTTP запросами, делая некоторую сложную бизнес логику, и затем подписывать некоторые сообщения с помощью HMAC. Поскольку он имеет сложную логику приложения, на сервере может быть незамеченный эксплойт для удаленного чтения файлов, который может предоставить злоумышленнику доступ к закрытому ключу.

Это может быть разделенный на два процесса в двух контейнерах: frontend контейнер, который обрабатывает пользовательское взаимодействие и бизнес логику, но который не имеет доступа к приватному ключу; и контейнер для подписи, который имеет доступ к приватному ключу и отвечает запросам простой подписи из frontend (для примера, через сеть localhost).

С таким разделенным подходом, теперь злоумышленнику приходится обманом заставить сервер приложений выполнить что-то довольно произвольное, что может оказаться сложнее, чем заставить его прочитать файл.
### Альтернативы Secret объекта

Прежде чем использовать Secret объект для защиты конфиденциальной информации, вы можете выбрать из альтернатив.

Здесь некоторые ваши варианты:
- Если ваши облачные компоненты в аутентификации в другом приложении и вы знаете, что оно запущено в пределах этого же Kubernetes кластера, вы можете использовать [[Authenticating#Service account tokens|ServiceAccount]] и его токен для идентификации вашего клиента.
- Есть сторонние инструменты, которые вы можете запустить в пределах или снаружи вашего кластера, которые будут управлять чувствительными данными. Для примера, служба к которой [[Pod]] получают доступ через HTTPS, которая раскрывает Secret объект если клиент корректно прошел аутентификацию (для примера, с [[Authenticating#Service account tokens|ServiceAccount]])
- Для аутентификации, вы можете реализовать собственный подписчик для сертификатов X.509 и использовать [[Certificates and Certificate Signing Requests|CertificateSigningRequests]], что бы позволить собственному подписчику выдавать сертификаты для [[Pod]] который в них нуждаются
- Вы можете использовать [[Device Plugins]] для предоставления аппаратного обеспечения для шифрования на локальной [[Nodes]] для указанных [[Pod]]. Для примера, мы можете запланировать доверенные [[Pod]] на узлах, которые предоставляют  Trusted Platform Module, настроенный вне группы.

Вы можете также объединять два или более этих варианта, включая варианты использования сами Secret объекты.

Для примера: реализовав (или развернув) [[Operator pattern|operator]] (специальный контроллер используемый для управления пользовательскими ресурсами) извлекающий короткоживущие токены сессии из внешних сервисов и затем создавая Secret объект основанный на этих короткоживущих токенах сессии. [[Pod]] работающие в вашем кластере могут использовать токены сессии, а  [[Operator pattern|operator]] гарантирует, что они верны. Это разделение гарантирует, что вы можете запустить [[Pod]], которые не подозревают о точных механизмах для издания и обновления этих токенов сессии.
## Типы Secret

При создании Secret объекта, вы можете указать его тип используя поле `type` Secret объекта или определенный аналог флага команды `kubectl` в терминале (если доступно). Тип Secret объекта используется для облегчения программной обработки данных Secret объекта.

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

Вы можете определить и использовать ваш собственный тип Secret объекта назначив не пустую строку как значение `type` для Secret объекта (пустая строка будет обработана как тип `Opaque`). 

Kubernetes не навязывает какие-либо ограничения на имя типа. Однако, если вы используете один из встроенных типов, вы должны соответствовать всем требованиям, определенным для этого типа.

Если вы определяете тип Secret объекта, который будет доступен для публичного использования, следуйте соглашению и структурируйте тип Secret объекта таким образом, что бы перед именем типа было ваше доменное имея , разделенное `/`. Для примера: `cloud-hosting-example.net/cloud-api-credentioals`.
### Secrets объект Opaque

`Opaque` стандартный тип Secret объекта если вы явно не указали тип в манифесте Secret объекта. Когда вы создаете Secret объект используя `kubectl`, вы должны использовать подкоманду `generic` для  обозначения `Opaque` типа. Для примера, следующая команда создает пустой Secret объект типа `Opaque`:

```shell
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

Вывод выглядит как:

``` shell
NAME           TYPE     DATA   AGE
empty-secret   Opaque   0      2m6s
```

Колонка `DATA` показывает количество элементов данных в Secret объекте. В этом случае, `0` означает, что вы создали пустой Secret объект.
### Secrets объект ServiceAccount токена

Тип Secret объекта `kubernetes.io/service-account-token` используется для хранения токена учетных данных идентифицирующий [[Authenticating#Service account tokens|ServiceAccount]]. Это устаревший механизм, который предоставляет долго живущие учетные данные [[Authenticating#Service account tokens|ServiceAccount]] для [[Pod]].

В Kubernetes v1.22 и позднее, рекомендуемый подход - получение короткоживущих, автоматически сменяющийся токен [[Authenticating#Service account tokens|ServiceAccount]] используемый вместо [[TokenRequest]]. Вы можете получить эти короткоживущие токены использующие следующие методы:
- Вызывает API `TokenRequest` напрямую или используя API клиент например как `kubectl`. Для примера: вы можете использовать команду [`kubectl create token`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-token-em-).
- Запросите смонтированный токен в [[Managing Service Accounts#Bound service account token volume mechanism|projected volume]] вашего [[Pod]] манифеста. Kubernetes создает токен и монтирует его в [[Pod]]. Токен автоматически признается не действительным когда [[Pod]] в который он смонтирован удален. Подробности смотри в [[Configure Service Accounts for Pods#Launch a Pod using service account token projection|Запуск Pod используя  service account token проекцию токена сервисного аккаунта]] 

> [!NOTE]
> Вы должны создать ServiceAccount токен Secret, только если вы не можете использовать API `TokenRequest` для получения токена и угрозы безопасности, связанной с сохранением учетных данных токена, срок действия которых не истекает в API объекте доступном вам для чтения.

Когда вы используете тип Secret объекта, вам необходимо убедиться что аннотация `kubernetes.io/service-account.name` установлена на существующее имя [[Authenticating#Service account tokens|ServiceAccount]]. Если вы создаете и [[Authenticating#Service account tokens|ServiceAccount]] и Secret объекты, вы должны создать первым объектом [[Authenticating#Service account tokens|ServiceAccount]]. 

После создания Secret объекта, Kubernetes [[Controllers|контроллер]] заполнит некоторые другие поля аннотацией `kubernetes.io/service-account.uid` и `token` ключ в поле `data`, который заполняется токеном аутентификации.

Для следующем примере конфигурация заявляет [[Authenticating#Service account tokens|ServiceAccount]] токен в Secret объекте

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

После создания Secret объекта дождитесь, пока Kubernetes введет ключ `token` в поле `data`.

Смотрите документацию [[Service Account]] для большей информации о том, как работает [[Service Account]]. Вы так же можете проверить [[Pod]] поля `automountServiceAccountToken` и `serviceAccountName` для информации о том, как ссылаться на учетные данные [[Service Account]] из модулей. 
### Secrets объект Docker конфигурации 

Если вы создаете Secret объект для хранения учетных данных для доступа к реестру образов контейнеров, вы должны использовать один из следующих значений `type` для этого Secret объекта:
- `kubernetes.io/dockercfg`:  сохраните стерилизованный файл  `~/.dockercfg`, который является устаревшим форматом для настройки командной строки Docker. Поле Secret объекта `data` содержит ключ `.dockercfg` значение которого это контент файла `~/.dockercfg` в закодированном base64 формата.
- `kubernetes.io/dockerconfigjson`: сохраните стерилизованный JSON, которые следует таким же правилам форматирования как и файл `~/.docker/config.json`, который является новым форматом для `~/.dockercfg`. Поле Secret объекта  `data` должно содержать ключ `.dockerconfigjson`, для которого значение является контентом закодированного формата base64  `~/.docker/config.json` файла.

Ниже пример для Secret объекта типа `kubernetes.io/dockercfg`:

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

Когда вы создаете Docker конфигурацию используя манифест Secret объекта , API сервер проверяет будь то ожидаемый существующий ключ поля `data` и проверяет его, если предоставляемое  значение может быть разобрано как корректный JSON. Сервер API не проверяет, действительно ли JSON является файлом конфигурации Docker.

Вы так же можете использовать `kubectl` для создания Secret объекта доступа к реестру контейнера,  когда у вас нет файла конфигурации Docker:

```shell
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

Команда создает Secret объект `kubernetes.io/dockerconfigjson` типа.

Для извлечения поля `.data.dockerconfigjson` из этого нового Secret объекта и декодирования данных:

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
> Значение `auth` закодировано в base64; это скрытно, но не секретно. Кто угодно может прочитать этот Secret объект может узнать токен владельца доступа к реестру.
> 
> Рекомендуется использование [[Configure a kubelet image credential provider|поставщиков учетных данных]] для динамического и надежного получения конфиденциальных данных по требованию.
 
### Secret объект базовой аутентификации

Тип `kubernetes.io/basic-auth` предоставлен для хранения учетных данных необходимых для базовой аутентификации. При использовании этого типа Secret объекта, поле `data` может должен содержать один из следующих двух ключей:
- `username`: Имя пользователя для аутентификации
- `password`: Пароль или токен для аутентификации

Оба значения для выше указанных ключей закодированные base64 строки. Вы можете альтернативно предоставить не кодированные данные используя `stringData` в манифесте Secret объекта.

Следующий манифест это пример Secret объекта для базовой аутентификации:

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
> Поле `strigData` для Secret объекта не работает хорошо с применением на стороне сервера

Базовый аутентификационный тип Secret объекта предоставлен только для удобства. Вы можете создать тип [[#Opaque Secrets|opaque]]  для учетных данных используемых при базовой аутентификации. Однако, использование заданного и публичного типа Secret объекта (`kubernetes.io/basic-auth`) помогает другим людям понимать цель вашего Secret объекта и установить соглашение какие имена ключей ожидать. Kubernetes API проверяет какие обязательные ключи установлены для этого типа Secret объекта.
### Secrets объект SSH аутентификации 

Встроенный тип `kubernetes.io/ssh-auth` предоставлен для хранения данных используемых в SSH аутентификации. При использовании этого типа Secret объекта, у вас будет определена `ssh-privatekey` пара ключ-значение в поле `data` (или `stringData`) как учетные данные SSH для использования.

Следующий манифест это пример Secret объекта используемого для SSH публичного/приватного ключа аутентификации:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  # данные сокращены в этом примере
  ssh-privatekey: |
    UG91cmluZzYlRW1vdGljb24lU2N1YmE=    
```

Тип Secret объекта SSH аутентификации предоставляется только для удобства. Вы можете создать тип [[#Opaque Secrets|opaque]] для учетных данных используемых при SSH аутентификации. Однако использование заданного и публичного типа Secret объекта (`kubernetes.io/ssh-auth`) поможет другим людям понять цели вашего Secret объекта и установить соглашение какие имена ключей ожидать. Kubernetes API проверяет какие обязательные ключи установлены для этого типа Secret объекта.

> [!CAUTION]
> SSH приватные ключи не создают доверительное общение между SSH клиентом и сервером сами по себе. Для предотвращения атак типа "man in the middle" необходимы дополнительные средства установления доверия, такие как файл `known_hosts`, добавленный в [[ConfigMap]].
### Secrets объект TLS 

Тип Secret объекта `kubernetes.io/tls` для хранения сертификата и связанного с ним ключа, который обычно используется для TLS.

Одно из распространенных применений TLS Secret объекта - настройка шифрования транзита для [[Ingress]] объекта, но вы можете также использовать его с другими ресурсами или напрямую в вашем workload. При использовании этого типа объекта Secret ключи `tls.key` и `tls.crt` должны быть предоставлены в поле `data` (или `stringData`) конфигурации Secret объекта, хотя API сервер не проверяет значения каждого ключа.

Как альтернативу для использования `stringData` вы можете использовать поле `data` для предоставления зашифрованных base64 сертификата и приватного ключа. 

Для более подробной информации смотрите [[#Constraints on Secret names and data]].

Следующий YAML содержит пример конфигурации для TLS Secret объекта:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # значения зашифрованы base64, который скрывает их но НЕ предоставляет какого-либо полезного уровня конфиденцальности
  tls.crt: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUNVakNDQWJzQ0FnMytNQTBHQ1NxR1NJYjNE
    UUVCQlFVQU1JR2JNUXN3Q1FZRFZRUUdFd0pLVURFT01Bd0cKQTFVRUNCTUZWRzlyZVc4eEVEQU9C
    Z05WQkFjVEIwTm9kVzh0YTNVeEVUQVBCZ05WQkFvVENFWnlZVzVyTkVSRQpNUmd3RmdZRFZRUUxF
    dzlYWldKRFpYSjBJRk4xY0hCdmNuUXhHREFXQmdOVkJBTVREMFp5WVc1ck5FUkVJRmRsCllpQkRR
    VEVqTUNFR0NTcUdTSWIzRFFFSkFSWVVjM1Z3Y0c5eWRFQm1jbUZ1YXpSa1pDNWpiMjB3SGhjTk1U
    TXcKTVRFeE1EUTFNVE01V2hjTk1UZ3dNVEV3TURRMU1UTTVXakJMTVFzd0NRWURWUVFHREFKS1VE
    RVBNQTBHQTFVRQpDQXdHWEZSdmEzbHZNUkV3RHdZRFZRUUtEQWhHY21GdWF6UkVSREVZTUJZR0Ex
    VUVBd3dQZDNkM0xtVjRZVzF3CmJHVXVZMjl0TUlHYU1BMEdDU3FHU0liM0RRRUJBUVVBQTRHSUFE
    Q0JoQUo5WThFaUhmeHhNL25PbjJTbkkxWHgKRHdPdEJEVDFKRjBReTliMVlKanV2YjdjaTEwZjVN
    Vm1UQllqMUZTVWZNOU1vejJDVVFZdW4yRFljV29IcFA4ZQpqSG1BUFVrNVd5cDJRN1ArMjh1bklI
    QkphVGZlQ09PekZSUFY2MEdTWWUzNmFScG04L3dVVm16eGFLOGtCOWVaCmhPN3F1TjdtSWQxL2pW
    cTNKODhDQXdFQUFUQU5CZ2txaGtpRzl3MEJBUVVGQUFPQmdRQU1meTQzeE15OHh3QTUKVjF2T2NS
    OEtyNWNaSXdtbFhCUU8xeFEzazlxSGtyNFlUY1JxTVQ5WjVKTm1rWHYxK2VSaGcwTi9WMW5NUTRZ
    RgpnWXcxbnlESnBnOTduZUV4VzQyeXVlMFlHSDYyV1hYUUhyOVNVREgrRlowVnQvRGZsdklVTWRj
    UUFEZjM4aU9zCjlQbG1kb3YrcE0vNCs5a1h5aDhSUEkzZXZ6OS9NQT09Ci0tLS0tRU5EIENFUlRJ
    RklDQVRFLS0tLS0K    
  # В этом примере, данные ключ не настоящий зашифрованный PEM приватный ключ
  tls.key: |
    RXhhbXBsZSBkYXRhIGZvciB0aGUgVExTIGNydCBmaWVsZA==    
```

Тип TLS Secret объекта предоставляет только удобство. Вы можете создать тип [[#Opaque Secrets|opaque]] для учетных данных используемых при TLS аутентификации. Однако, использования заданного и публичного типа Secret объекта (`kubernetes.io/tls`) помогает убедиться в согласованности формата Secret объекта в вашем проекте. API сервер проверяет обязательные ключи установленные в это тип Secret объекта.

Для создания TLS Secret объекта используя kubectl, используйте подкоманду `tls`

```shell
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

Публичный/приватный пары ключей должны существовать заранее. Публичный ключ сертификата `--cert` должен быть зашифрован .PEM и должен совпадать с предоставленным приватным ключом `--key`.
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


