Года с 2016 для своих собственных хобби и для помощи чужим некоммерческим проектам использую GCP, но Google признался в шовинизме, русофобии и ввел дискриминационные ограничения на людей, в число которых вхожу я. Даже то, что я украинец, не имеет значения, ведь под ненавистными русскими они подразумевают всю культуру, всех славян, всех пользователей кириллицы.

# Переезд с Google Cloud (GCP) на Yandex Cloud (YC)

Здесь будет обзор примера примера переезда, а также заметки, ответы на вопросы, которые могут возникать в процессе.

## Disclaimer

Я не специалист ни по переносу облаков, ни по Яндекс Облаку, ни по AWS-совместимым инструментам, так что советы приветствуются.

![dog](https://user-images.githubusercontent.com/2870363/159172571-361753a7-3516-4a0b-9726-c139c0ba5c54.jpg)

## Тарификация и Free Tier

> 17 марта 2022: <Нарек Татевосян> Free Tier для Compute не планируется.

## Сервисы

### Storage -> Object Storage

https://cloud.google.com/storage/pricing  
https://cloud.yandex.ru/docs/storage/pricing

|                | Google      | Yandex  |
| -------------- | :---------: | :-----: |
| Free GB-months | 5           | 1       |

Т.к. переносить я буду не все, делать я это буду в два этапа, поэтому единого скрипта не будет.

Отсортировать бакеты по размеру: `$ gsutil du -s gs://* | sort -nr`  
Скачать: `$ gsutil mv gs://.../* ~/_/...`  

https://cloud.google.com/storage/docs/gsutil/commands/mv#renaming-groups-of-objects  
Перед `mv` можно добавить `-m`, чтобы качало параллельно.  
`*` в src аргументе gsutil не может указывать на директории, только на файлы.  
Если использовать не `gsutil mv`, а `gsutil rsync`, то можно обойтись без звездочки:  
```
$ mkdir ~/_/my_bucket
$ gsutil -m rsync -r gs://my_bucket ~/_/my_bucket/
$ gsutil -m rm -r gs://my_bucket/*  # м.б. удаление бакета целиком затратит меньше платных операций
```

Деплой в Run и Functions захламляет бакет с названием типа `[<region>.]artifacts.<projectname>.appspot.com` -- он нужен для того, чтоб восстанавливать образы прошлых версий. Чистить его [рекомендуют](https://stackoverflow.com/a/67693952/322020) не вручную, а через https://console.cloud.google.com/gcr хотя найти там что-либо проблематично -- я нашел файлы, совпадающие по дате, удалил, а в бакете они остались. Не знаю, почему нет ни обратных ссылок, ни поиска в реестре по имени/хешу. Единственное наверное, что можно еще сделать -- скачать образ, как-то скормить докеру и посмотреть, что внутри.

Чтобы использовать бакет для статического сайта, имена файлов `index.html` и `404.html` нужно указать явно в настройках бакета.

За 10 лет AWS так лучше не стал, но почему-то именно с ним в YC стремятся быть совместимыми. Официальный докер-образ AWS CLI концептуально не имеет ничего, кроме самой тулзы. Т.е. нельзя так просто перейти с gsutil на aws-cli, например, для реализации такого Github Action https://github.com/Nakilon/git-to-gcs/blob/master/Dockerfile потому что в образ нельзя доустановить ruby. Однако вместо aws-cli можно воспользоваться rclone, который написан на Go и потому тривиально помещается в любое окружение, поэтому аналог упомянутого инструмента теперь здесь: https://github.com/Nakilon/git-to-os/tree/0.0.0

Для программной заливки в бакет на Ruby достаточно `net/http` и гема `aws-sigv4`.

### Run -> Serverless Containers

https://cloud.google.com/run/pricing  
https://cloud.yandex.ru/docs/serverless-containers/pricing

|                   | Google      | Yandex  |
| ----------------- | :---------: | :-----: |
| Free vCPU hour    | 50          | 5       |
| Free RAM GiB*hour | 100         | 10      |

Деплоить локальный образ как Serverless Container сразу нельзя. Порядок действий следующий:
* создать сам Реестр
* создать Контейнер: `yc serverless container create --name <container_name>`
* предварительно включив руками [Docker Credential helper](https://cloud.yandex.ru/docs/container-registry/operations/authentication#cred-helper), запушить образ в Container Registry, см. https://cloud.yandex.ru/docs/container-registry/operations/docker-image/docker-image-push
* создать Ревизию в Контейнере -- это удобней через браузер

Для создания Триггера по таймеру и непосредственно запуска контейнера нужно сделать два Сервис аккаунта, которым не нужно создавать ключи. Не спрашивайте зачем.

Во время этих ручных манипуляций можно совершить такую ошибку, что если запушить новый образ, а потом еще раз задеплоить, то использоваться будет по прежнему старый контейнер. Потому что нужно `:latest` к урлу в команде деплоя, а сам он там не подразумевается, что неочевидно для пользователя GCP, который может без лишнего шага ручной загрузки в любой момент запушить локальный образ, и docker сам подхватит именно локальный `:latest` тег.

#### пример инструкции

```none
build and test
  $ yc container registry list
  $ docker run --rm -ti -e PORT=8001 -p 8001:8001 cr.yandex/<registry ID>/my-container-name
  $ curl http://127.0.0.1:8001/
  $ docker build -t cr.yandex/.../my-container-name . < Dockerfile
initial deploy:
  $ docker push cr.yandex/.../my-container-name
  $ yc serverless container create --name my-container-name
  # создать service account (SA) с ролью container-registry.images.puller на репозиторий (выдача роли не на уровнее репозитория, а на уровне проекта ничего не дает)
  $ yc iam service-account list | grep my-container-name
  $ yc serverless container revision deploy \
    --container-name my-container-name \
    --image cr.yandex/.../my-container-name \
    --service-account-id <SA id> \
    --concurrency 1
  # --execution-timeout 5s \
  # создать Триггер (кнопка СЛЕВА в Serverless Containers)

update:
  $ docker build ...
  $ docker push ...
  $ yq serverless container revision deploy ...
```

### Functions -> Cloud Functions

Языков меньще. Ruby нет.

### Scheduler -> [триггер "Таймер"](https://cloud.yandex.ru/docs/functions/concepts/trigger/timer)

Это не отдельный сервис, а ф-ционал Serverless.

cron-синтаксис у него такой, что `0 0 * * * *` писать нельзя, а надо `0 0 ? * * *`.  

### Datastore -> ?

### Pub/Sub -> ?

### Compute Engine -> Compute Cloud

### Logging -> Logging?

### Error Reporting -> ?

### Monitoring -> ?
