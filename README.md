### Назначение
Развертывание менеджера паролей Passbolt, MariaDB и Reverse Proxy Traefik в сегменте сети без публичного доступа с автоматическим получением X509 сертификатов Let's Encrypt через DNS валидацию, используя API Cloudflare

#### Назначение Passbolt
Основное приложение. Менеджер паролей

#### Назначение Traefik
Reverse Proxy. Терминирует TLS, автоматически получает сертификат сервера в Let's Encrypt через DNS валидацию. Для создания TXT записи для DNS валидации используется API Cloudflare.

#### Назначение MariaDB
С образом MariaDB создается 2 контейнера:
1. mysql. СУБД. Хранит базу данных с конфигурацией и паролями Passbolt
2. mysqlbackup. Используется для автоматического бекапа БД Passbolt

### Требования для установки
1. Сервер с установленным Docker Compose 
2. Свободные порты 80/TCP и 443/TCP на сервере
3. Доступ с сервера в Интернет для ACME. Доступ к серверу из Интернет опционален.
4. Доменное имя в DNS зоне, делегированной Cloudflare (при использовании внутреннего DNS, создавать A запись в DNS зоне на публичном DNS сервере не требуется)
5. API Token (не API Key) Cloudflare для зоны с правами Zone:Read, DNS:Edit для зоны, используемой в доменном имени Passbolt

### Установка
1. Клонировать репозиторий в /opt/passbolt 

```
sudo mkdir /opt/passbolt -p && \
sudo chown ${USER}:${USER} /opt/passbolt && \
git clone https://github.com/nd4y/passbolt-docker-compose.git /opt/passbolt && \
cd /opt/passbolt
```
2. Скопировать /opt/passbolt/.env.example в /opt/passbolt/.env
3. В файле /opt/passbolt/.env сконфигурировать значение обязательных переменных:
- TRAEFIK_ACME_EMAIL
- PASSBOLT_HOSTNAME
- PASSBOLT_EMAIL_DEFAULT_FROM
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_USERNAME
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_HOST
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_PORT
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_TLS

Описание переменных указано в .env в комментариях
   
4. Создать каталог /opt/passbolt/mounts/mysqlbackup/backups `mkdir -p /opt/passbolt/mounts/mysqlbackup/backups -p` для хранения резервных копий базы данных.
5. Создать каталог /opt/passbolt/secrets `mkdir -p /opt/passbolt/secrets -p` для хранения Docker Secrets
6. Записать желаемый пароль к СУБД в /opt/passbolt/secrets/mysql_password.secret
```
echo '$uper$ecretPa$$w0rd' > /opt/passbolt/secrets/mysql_password.secret'
```
7. Записать пароль к SMTP серверу для отправки почты Passbolt в /opt/passbolt/secrets/passbolt_email_transport_default_password.secret
```
echo '$uper$ecretPa$$w0rd' > /opt/passbolt/secrets/passbolt_email_transport_default_password.secret'
```
8. Записать API Token Cloudflare в /opt/passbolt/secrets/cf_api_token.secret
```
echo 'MvojUnak4rx234576544fbCxmZ2kCvUktlz' > /opt/passbolt/secrets/cf_api_token.secret'
```
9. Запустить docker-compose `cd /opt/passbolt && docker compose up -d`
10. Создать в Passbolt пользователя с правами администратора, указав ключи
- u - Email/Логин
- f - Имя
- l - Фамилия
```
docker compose -f docker-compose.yml \
exec passbolt su -m -c "source /etc/environment && \
/usr/share/php/passbolt/bin/cake \
passbolt register_user \
-u admin@example.org \
-f Admin \
-l Admin \
-r admin" -s /bin/bash www-data
```
11.  Перейти по выведенной в терминал ссылке и пройти регистрацию
12.  Экспортировать GPG ключи https://www.passbolt.com/docs/hosting/backup/from-docker/#2-the-server-public-and-private-keys и сохранить их отдельно от инсталляции Passbolt. GPG ключи понадобятся при восстановлении бекапа Passbolt в другом окружении
```
mkdir -p passbolt-gpg-export
docker compose cp passbolt:/etc/passbolt/gpg/serverkey_private.asc ./passbolt-gpg-export/
docker compose cp passbolt:/etc/passbolt/gpg/serverkey.asc ./passbolt-gpg-export/
```


### Резервное копирование базы данных

Резервное копирование выполняется контейнером mysqlbackup. 
Конфигурацией резервного копирования можно управлять с помощью переменных в .env :
- MYSQLBACKUP_BACKUPS_PATH
- MYSQLBACKUP_NUMBER_OF_RECOVERY_POINTS
- MYSQLBACKUP_INIT_SLEEP
- MYSQLBACKUP_INTERVAL

По умолчанию: каждые 24 часа с момента запуска контейнера mysqlbackup создается новая резервная копия с помощью mysqldump, которая сжимается в gz архив. После создания резервной копии, выполняется удаление всех существующих резервных копий, кроме последних 30. То есть хранится 30 точек восстановления.

Описание переменных указано в .env в комментариях



##### Полезные команды в контейнере Passbolt
```
su -m -c "source /etc/environment && /usr/share/php/passbolt/bin/cake passbolt send_test_email --recipient=admin@example.org" -s /bin/bash www-data
su -m -c "source /etc/environment && /usr/share/php/passbolt/bin/cake passbolt show_queued_emails" -s /bin/bash www-data
su -m -c "source /etc/environment && /usr/share/php/passbolt/bin/cake passbolt healthcheck" -s /bin/bash www-data
```