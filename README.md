# Docker Monitoring (Echo + Cobra + nginx + React)
Сервис для мониторинга информации о состоянии контейнеров.

## Запуск
1. Склонируйте репозиторий (вместе с модулями).
```
git clone --recurse-submodules git@github.com:spanwalla/docker-monitoring.git
cd docker-monitoring
```
Альтернативный вариант, если есть проблемы с SSH-ключом:
```
git clone --recurse-submodules https://github.com/spanwalla/docker-monitoring
cd docker-monitoring
```
2. Создайте файл `.env` в корневом каталоге проекта, вы можете взять за основу файл [.env.example](.env.example)
3. Для запуска контейнеров выполните команду
```
docker-compose up --build -d
```
4. Для остановки используйте команду
```
docker-compose down --remove-orphans
```

Сервис будет доступен на портах **8080** (backend) и **35822** (frontend).

После первого запуска нужно подождать примерно две минуты.

В случае ошибок стоит посмотреть логи контейнеров (`docker logs <имя контейнера>`), а также лог запросов к бэкенд-серверу в каталоге `backend/logs/requests.log`.

## Спорные вопросы и их решения
1. Было не совсем понятно, что именно подразумевается под фразой "пингануть контейнер". Cделал традиционный ping по IP-адресу контейнера, но так как ICMP обычно отключён, решил дополнительно взять значения полей `state` и `status` из информации о контейнере.
2. В задании не говорилось про авторизацию пингеров, но мне показалось логичным добавить это, как минимум для идентификации тех, кто посылает отчёты.

## Описание решения
- Frontend: написан на React, также используется `Vite` и `antd`. Собирается и раздаётся с помощью `nginx`. Посылает запросы на Backend с интервалом пять секунд. Информация обновляется без перезагрузки страницы.
- Backend: написан на Go с использованием Echo и многих других пакетов. Написан с соблюдением принципов чистой архитектуры. Реализована JWT-авторизация, graceful shutdown, а также доступна Swagger-документация по маршруту `/swagger/index.html`.
  - Сохранение отчётов реализовано с использованием сервиса очередей **RabbitMQ**, работает это так: пингер посылает REST-запрос к API, сервер возвращает `202 Accepted` и добавляет отчёт в очередь, откуда её берёт и помещает в БД параллельно работающая горутина.
- Pinger: написан как консольная утилита, по замыслу должна запускаться на отслеживаемой хост-машине и собирать информацию о контейнерах. Здесь для демонстрации запускается вместе со всеми остальными сервисами, что, соответственно, означает, что она будет собирать информацию о backend, frontend, postgres, rabbitmq и о самом себе.
  - Варианты запуска на хост-машине разные: использование бинарного файла (в `Makefile` предусмотрен сценарий сборки исполняемых файлов для основных ОС), запуск в Docker-контейнере (для этого нужны особые настройки `network_mode: host` и монтирование `/var/run/docker.sock` с хост-машины, а также установка через `go install`. На мой взгляд, наиболее предпочтительный способ – использование `go install`.
  - Предусмотрен graceful shutdown.
  - Интервал отправки отчётов задаётся через конфиг (образец в [config.yaml](pinger/config/config.yaml)) в формате cron-строки, если путь к конфиг-файлу не указан, то будет использоваться конфиг из `$HOME/.docker-pinger.yaml` (если его нет, то он будет создан со стандартными значениями).

## Демонстрация
1. Без данных.

![Screenshot 2025-02-08 110139](https://github.com/user-attachments/assets/91b4db32-8d38-421e-8663-6b8abee2a2ca)

2. Просмотр информации от нескольких пингеров (реализован в виде расширяемой таблицы).

![Screenshot 2025-02-08 112056](https://github.com/user-attachments/assets/000e356e-daa5-499c-b167-1775cd3bfab6)

3. Запрос на сохранение отчёта в Postman.

![Screenshot 2025-02-08 115616](https://github.com/user-attachments/assets/ef4a6520-2f86-4a9f-bcbb-b8ff13ca5800)

4. Интерактивность

![tocropandgif-ezgif com-video-to-gif-converter](https://github.com/user-attachments/assets/2edf24e9-b105-458b-ab11-62c528444234)

5. Обновление информации без перезагрузки

![tocropandgif-ezgif com-video-to-gif-converter (1)](https://github.com/user-attachments/assets/f82198a9-8b78-45bb-8636-b27160ed8d80)
