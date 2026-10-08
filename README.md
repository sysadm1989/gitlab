# Self-hosted GitLab CE

Плейбук ставит GitLab CE на Ubuntu 26.04 и закрывает инстанс, который смотрит в интернет. Обязательные апрувы вида «1 от security и 1 от devops» GitLab CE сам не умеет. На этой ВМ остаётся только `pre-receive` hook. Вызов `https://idp.nemero.corp/approvals/check` включается флагом `gitlab_approval_check_enabled`. Сейчас он выключен.

## Что будет наружу

Открыты порты 22, 80 и 443. Регистрация выключена, проекты по умолчанию private, публичная видимость запрещена. SSH только по ключу.

CrowdSec стоит как сетевой фильтр (firewall bouncer на nftables), а не как модуль nginx. У пакета GitLab свой nginx, и чужой модуль сломается на следующем обновлении. Движок берётся из официального репозитория CrowdSec: в Ubuntu 26.04 свой пакет 1.4.6 для этой задачи старый. Локальные сценарии банят перебор SSH, типовые HTTP-атаки и подбор формы входа. Форма `/users/sign_in` отвечает кодом 200, поэтому для неё есть отдельный сценарий. Общий список адресов CrowdSec подтягивается через CAPI. Ключ консоли необязателен. Как смотреть баны, снимать их и менять белый список, написано в [crowdsec.md](crowdsec.md).

Лимиты входа в самом GitLab остаются включёнными.

## Перед запуском

- DNS имени из `gitlab_external_url` указывает на ВМ, порты 22/80/443 доступны снаружи. Let's Encrypt проверяет порт 80.
- На ВМ уже есть пользователь из `ssh_allow_users`, и вход по ключу работает. Плейбук выключает пароль SSH.
- В `crowdsec_whitelist_cidrs` указана сеть, с которой вы администрируете хост.

В `inventories/example/group_vars/gitlab/vars.yml` замените домен, почту, SMTP и имя бакета. Версия пакета зафиксирована (`gitlab_ce_version`) и поставлена на hold.

Пароли читаются из HashiCorp Vault в момент запуска, в git их нет. KV v2: `https://10.140.2.30:8200`, mount `platforms`, путь `prod/gitlab`. Поля:

- `gitlab_root_password` — пароль локального `root`, не короче 12 символов
- `gitlab_smtp_password` — пароль ящика SMTP
- `backup_s3_access_key` и `backup_s3_secret_key` — доступ к бакету бэкапов
- `keycloak_client_secret` — секрет уже созданного клиента `gitlab`

Секрет читает сама ВМ: с ноутбука через VPN порт 8200 открывается, но TLS-ответ Vault не доходит.

```bash
cd ansible
export ANSIBLE_HASHI_VAULT_TOKEN=...
ansible-playbook site.yml
```

Старый `vault.yml` из ansible-vault в этот каталог класть нельзя: Ansible попытается его расшифровать до обращения в HashiCorp Vault.

Люди входят только через SSO `https://sso.nemero.online/realms/gitlab`. Форма пароля выключена. Имя в GitLab берётся из `preferred_username`. Пароль локального `root` лежит в Vault в поле `gitlab_root_password` и нужен только для `gitlab-rails` на сервере. Учётным записям из SSO второй фактор GitLab не нужен. Участник группы Keycloak `a_gitlab_admins` при входе становится администратором: в ID token или userinfo должен быть claim `groups`. Если группы в claim нет, права администратора снимаются при следующем входе. Локальный `root` от этой группы не зависит. Секрет клиента `gitlab` лежит в Vault в поле `keycloak_client_secret`, в git его нет. LDAP пока выключен, плейбук realm не переписывает. Чтобы снова открыть форму пароля, поставьте `gitlab_local_login_enabled: true` и прогоните плейбук.

## Бэкап

Таймер `gitlab-backup.timer` срабатывает в 02:15, с разбросом до 20 минут. `gitlab-backup create` пишет архив данных и сразу отправляет его в бакет. Имя начинается с даты сервера, дальше Unix-время в секундах и версия GitLab: `2026_10_08_1791447230_19.4.1_gitlab_backup.tar`. Рядом, в `config/<тот же префикс>-config.tgz`, лежит второе, что нужно для подъёма: `gitlab-secrets.json`, `gitlab.rb`, пароль SMTP и ключи бакета. После успешной отправки локальный tar с диска ВМ удаляется.

Объекты в бакете удаляет lifecycle на 60 дней (`gitlab_backup_s3_retention_days`). Плейбук ставит это правило на весь бакет, включая `config/`. Яндекс применяет его примерно раз в сутки, поэтому объект исчезает вскоре после 60-го дня, не ровно в полночь. В консоли то же правило: Object Storage → бакет → «Жизненный цикл» → удалить объекты через 60 дней после загрузки, префикс пустой. Локальные копии на ВМ скрипт убирает раньше, через `gitlab_backup_keep_seconds` (сейчас 7 дней).

Если служба бэкапа завершилась с ошибкой, `gitlab-backup-alert.service` отправляет письмо на `gitlab_backup_alert_email` через тот же SMTP. В письме последние строки журнала этого запуска. Успешный бэкап писем не шлёт.

Проверка вручную:

```bash
sudo systemctl start gitlab-backup.service
```

Список копий:

```bash
aws s3 ls s3://gitlab-nemero-backups/ --endpoint-url https://storage.yandexcloud.net
aws s3 ls s3://gitlab-nemero-backups/config/ --endpoint-url https://storage.yandexcloud.net
```

## Восстановление

Нужны оба объекта с одним префиксом, например `2026_10_08_1791447230_19.4.1`. Версия в имени должна совпасть с установленным `gitlab-ce`: сначала эта версия, и только потом более новая.

1. Поднимите чистую ВМ и прогоните плейбук с той же `gitlab_ce_version`.
2. Скачайте архив и секреты:

```bash
aws s3 cp s3://gitlab-nemero-backups/2026_10_08_1791447230_19.4.1_gitlab_backup.tar /var/opt/gitlab/backups/ --endpoint-url https://storage.yandexcloud.net
aws s3 cp s3://gitlab-nemero-backups/config/2026_10_08_1791447230_19.4.1-config.tgz /root/ --endpoint-url https://storage.yandexcloud.net
```

3. Верните секреты на место до восстановления базы. `gitlab-secrets.json` из этого архива обязан остаться: без него не расшифруются CI-переменные и секреты двухфакторного входа. Плейбук этот файл не перезаписывает.

```bash
tar -xzf /root/2026_10_08_1791447230_19.4.1-config.tgz -C /etc/gitlab
chmod 600 /etc/gitlab/gitlab-secrets.json /etc/gitlab/smtp_password /etc/gitlab/backup_s3_access_key /etc/gitlab/backup_s3_secret /etc/gitlab/backup_s3_credentials
chown git:git /var/opt/gitlab/backups/2026_10_08_1791447230_19.4.1_gitlab_backup.tar
gitlab-ctl reconfigure
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
gitlab-backup restore BACKUP=2026_10_08_1791447230_19.4.1 force=yes
gitlab-ctl reconfigure
gitlab-ctl restart
```

4. Прогоните плейбук ещё раз. Он вернёт hook апрувов, CrowdSec и таймер бэкапа. Пароль `root` в базе будет из архива.

## Апрувы

На ВМ GitLab сервис политики не запускается. Hook `pre-receive` при `gitlab_approval_check_enabled: true` отправляет запись на `https://idp.nemero.corp/approvals/check`. Ответ `{"allow": true}` пропускает push. Любой другой ответ, обрыв или ошибка разбора отклоняют запись. Сейчас флаг `false`: hook есть, но внешний сервис не вызывается и push из-за него не блокируется. Включить вызов — поставить `true` и прогнать плейбук.

Группы участников и правила апрувов тоже живут на `idp.nemero.corp`. Что ему реализовать и что убрать в GitLab, написано в [idp-approvals.md](idp-approvals.md). Этот плейбук группы не создаёт и состав из Keycloak на ВМ не сверяет.

Плейбук не хранит правила апрувов и не поднимает для них сервис.

## Keycloak

Realm, клиент `gitlab` и федерация с AD настраиваются другим плейбуком. Этот только подключает GitLab к уже готовому issuer `https://sso.nemero.online/realms/gitlab`. Логин берётся из `preferred_username`. Секрет клиента читается из Vault, поле `keycloak_client_secret`.

В GitLab LDAP выключен. Группы из токена в группы GitLab сами не попадают. Членство задаёт `idp.nemero.corp`, не этот плейбук.

Вход только через SSO `https://sso.nemero.online`. Форма пароля выключена.

## Чего фильтр не делает

- Не подменяет встроенные approval rules Premium. Кнопка Approve остаётся штатной, запрет влития делает hook, который спрашивает `idp.nemero.corp`.
- Не ставит WAF внутрь nginx GitLab.
- Не проверяет теги и не смотрит на CODEOWNERS.
- Не банит форму входа, если парсер nginx не разобрал строку. Лимит GitLab на `/users/sign_in` при этом работает.
- Самоподписанный сертификат для интернета не используется. `gitlab_tls_mode: selfsigned` пускает только вместе с `gitlab_allow_selfsigned: true`.
