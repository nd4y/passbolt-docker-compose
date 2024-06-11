### Назначение
Развертывание менеджера паролей Passbolt, MariaDB и Reverse Proxy Traefik в сегменте сети без публичного доступа с автоматическим получением X509 сертификатов Let's Encrypt через DNS валидацию, используя API Cloudflare

#### Назначение Passbolt
Основное приложение. Менеджер паролей

#### Назначение Traefik
Reverse Proxy. Терминирует TLS, автоматически получает сертификат сервера в Let's Encrypt через DNS валидацию. Для создания TXT записи для DNS валидации используется API Cloudflare.

#### Назначение MariaDB
СУБД. Хранит базу данных с конфигурацией и паролями Passbolt

### Требования для установки
1. Сервер с установленным Docker Compose, 
2. Свободные порты 80/TCP и 443/TCP на сервере
3. Доступ с сервера в Интернет для ACME. Доступ к серверу из Интернет опционален.
4. Доменное имя в DNS зоне, делегированной Cloudflare (при использовании внутреннего DNS, создавать A запись в DNS зоне на публичном DNS сервере не требуется)
5. API Token (не API Key) Cloudflare с правами Zone:Read, DNS:Edit
### Установка
1. Клонировать репозиторий в /opt/passbolt 

```
sudo mkdir /opt/passbolt -p && \
sudo chown ${USER}:${USER} /opt/passbolt && \
git clone https://github.com/nd4y/passbolt-docker-compose.git /opt/passbolt && \
cd /opt/passbolt
```
2. Скопировать /opt/passbolt/.env.example в /opt/passbolt/.env
3. В файле /opt/passbolt/.env сконфигурировать значение обязатеных переменных:
- TRAEFIK_ACME_EMAIL
- PASSBOLT_HOSTNAME
- PASSBOLT_EMAIL_DEFAULT_FROM
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_USERNAME
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_HOST
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_PORT
- PASSBOLT_EMAIL_TRANSPORT_DEFAULT_TLS

Описание переменных указано в .env
4. Создать каталог /opt/passbolt/secrets -p `mkdir -p /opt/passbolt/secrets -p`
5. Записать желаемый пароль к СУБД в /opt/passbolt/secrets/mysql_password.secret
```
echo '$uper$ecretPa$$w0rd' > /opt/passbolt/secrets/mysql_password.secret'
```
6. Записать пароль к SMTP серверу для отправки почты Passbolt в /opt/passbolt/secrets/passbotl_email_transport_default_password.secret
```
echo '$uper$ecretPa$$w0rd' > /opt/passbolt/secrets/passbotl_email_transport_default_password.secret'
```
7. Записать API Token Cloudflare в /opt/passbolt/secrets/cf_api_token.secret
```
echo 'MvojUnak4rx234576544fbCxmZ2kCvUktlz' > /opt/passbolt/secrets/cf_api_token.secret'
```
8. Запустить docker-compose `cd /opt/passbolt && docker compose up -d`
9. Создать в Passbolt пользователя с правами администратора, указав ключи
- u - Email/Логин
- f - Имя
- l - Фамилия
```
docker compose -f docker-compose.yml \
exec passbolt su -m -c "/usr/share/php/passbolt/bin/cake \
passbolt register_user \
-u Admin@example.org \
-f Admin \
-l Admin \
-r admin" -s /bin/sh www-data
```
10. Перейти по выведенной в терминал ссылке и пройти регистрацию


##### Полезные команды в контейнере Passbolt
```
su -m -c "/usr/share/php/passbolt/bin/cake passbolt send_test_email --recipient=admin@example.org" -s /bin/sh www-data
su -m -c "/usr/share/php/passbolt/bin/cake passbolt show_queued_emails" -s /bin/sh www-data
su -m -c "/usr/share/php/passbolt/bin/cake passbolt healthcheck" -s /bin/sh www-data
```