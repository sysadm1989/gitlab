# Установка self-hosted GitLab CE

## Контекст

Нужен self-hosted GitLab Community Edition: свой инстанс на своей ВМ, не GitLab.com и не GitLab Dedicated. Установка и базовая настройка делаются одним Ansible-плейбуком. Повторный прогон на том же хосте идемпотентен и не затирает уже созданные проекты и репозитории.

Обязательные апрувы этот плейбук не описывает и не хранит. Их делает сервис `idp.nemero.corp`, см. `idp-approvals.md`.

## Цель

Плейбуком поднять self-hosted GitLab CE и включить базовую защиту веток.

## Допущения

- Одна ВМ с Ubuntu 26.04 LTS (Resolute), архитектура `amd64` или `arm64`. Доступ к GitLab по HTTPS.
- Пакет GitLab CE ставится из официального репозитория `ubuntu/resolute`. Нативные пакеты для 26.04 есть начиная с GitLab CE 19.3.0 (август 2026). Пакеты Ubuntu 24.04 (`noble`) на эту систему не ставим.
- Control node с Ansible может ходить на ВМ по SSH под пользователем с sudo.
- SSO, Container Registry, GitLab Pages и Mattermost в этот таск не входят.

## Поставка через Ansible

Один плейбук `site.yml` и пять ролей. Переменные инстанса — в inventory/`group_vars`. Пароли и токены — в `ansible-vault`.

- `hardening` — SSH по ключу, ufw (22/80/443), sysctl, security-обновления ОС. Пакет GitLab в автообновления не входит.
- `gitlab` — репозиторий `resolute`, пакет `gitlab-ce` не ниже 19.3.0, `gitlab.rb` (`external_url`, SMTP, расписание бэкапа), `gitlab-ctl reconfigure` только если конфиг изменился.
- `baseline` — через API инстанса: выключенная публичная регистрация, дефолтная защита веток для новых проектов, те же protected branches на уже существующих проектах.
- `approvals` — только `pre-receive` hook. При `gitlab_approval_check_enabled: true` он спрашивает `https://idp.nemero.corp/approvals/check`. Сейчас флаг выключен. Локальный сервис политики на ВМ не ставится.
- `crowdsec` — движок с официального репозитория и nftables firewall bouncer. Модуль в nginx не ставится: у Omnibus свой nginx, чужой lua-модуль сломается на обновлении.

Глобальный hook лежит в каталоге custom hooks Gitaly и действует на все репозитории. Отключить его из настроек проекта нельзя. Пока `gitlab_approval_check_enabled: false`, он запись не блокирует.

## Установка и базовая настройка

Роль `gitlab` и роль `baseline` приводят инстанс к такому состоянию:

- Текущий стабильный GitLab CE из репозитория `resolute`, не ниже 19.3.0. `external_url` на HTTPS. После прогона в `apt policy gitlab-ce` источник — `resolute`, версия ОС — 26.04.
- Публичная регистрация выключена.
- «Pipelines must succeed» и «All threads must be resolved» включены по умолчанию для новых проектов, где это доступно на Free.
- Ветка по умолчанию на уровне инстанса для новых проектов и для уже существующих:
  - Allowed to push: No one;
  - Allowed to merge: Maintainers;
  - Allowed to force push: выключено.
  - Первый коммит, который создаёт ветку в пустом проекте, разрешён. Следующие обновления этой ветки — только через merge request.
- SMTP и расписание `gitlab-backup`. Архив данных и `gitlab-secrets.json` уходят в S3. Плейбук проверяет, что таймер бэкапа стоит. Сбой бэкапа шлёт письмо на `gitlab_backup_alert_email`.

## Как подключается внешняя проверка

На ВМ GitLab остаётся один компонент. Его ставит роль `approvals`.

**Server hook `pre-receive`.** Срабатывает на запись в репозиторий, в том числе на Merge из UI. Если `gitlab_approval_check_enabled: true`, отправляет данные на `https://idp.nemero.corp/approvals/check`. Если внешний сервис не разрешил запись, hook завершается с ненулевым кодом и печатает его текст в stderr. Maintainer не может отключить hook из проекта. Пока флаг `false`, hook сразу пропускает запись и наружу не ходит.

Сервис, который читает апрувы и решает, выполнены ли правила, на этой ВМ не запускается. System hook на localhost плейбук снимает и заново не создаёт.

## Состав поставки

- Плейбук и роли `hardening`, `gitlab`, `baseline`, `approvals`, `crowdsec`.
- Пример inventory для одной ВМ и список переменных (`external_url`, SMTP).
- `pre-receive` hook на `https://idp.nemero.corp/approvals/check`. Локальный unit сервиса политики роль удаляет. Схема правил — в `idp-approvals.md`, не в этом плейбуке.
- Runbook на сценарии из раздела «Поставка через Ansible».

## Критерии приёмки

- `ansible-playbook site.yml` на чистой Ubuntu 26.04 ставит GitLab CE из репозитория `resolute` (версия не ниже 19.3.0). Инстанс открывается по HTTPS, регистрация снаружи выключена, расписание бэкапа создано.
- Повторный прогон завершается без ошибок и не удаляет проекты, группы и репозитории.
- Новый проект получает защищённую ветку по умолчанию: push никому, merge только Maintainer, без force push. Уже существующие проекты роль `baseline` приводит к тем же правилам.
- При `gitlab_approval_check_enabled: false` hook не отклоняет запись. При `true` отказ внешнего сервиса виден в тексте ошибки Merge, и Maintainer не выключает hook из настроек проекта.

## Защита публичного инстанса

Инстанс доступен из интернета. Это не значит, что репозитории публичные.

- Снаружи открыты только 22, 80 и 443. SSH только по ключу, root по паролю выключен.
- Регистрация выключена, новые проекты и группы по умолчанию private, уровень public запрещён.
- 2FA обязательна, пароль от 12 символов, сессия 8 часов, срок PAT ограничен.
- Registry, Pages, Mattermost, KAS и импорт с внешних источников выключены. Экспортеры GitLab слушают localhost.
- Вход и git по HTTP режутся лимитами GitLab. CrowdSec банит IP по SSH, типовым HTTP-атакам и подбору формы `/users/sign_in`.
- Сервис политики на ВМ GitLab не слушает. Локальный system hook на `127.0.0.1:8765` снимается.

Плейбук лежит в `ansible/`. Как запускать и что сознательно не закрыто, написано в `README.md`. Правила для внешнего сервиса — в `idp-approvals.md`.

## Вход через Keycloak

Люди входят по OIDC на `https://sso.nemero.online/realms/gitlab`, GitLab открыт как `https://git.nemero.online`. Форма пароля выключена (`gitlab_local_login_enabled: false`). Realm и федерацию Keycloak с AD настраивает другой плейбук. Здесь GitLab только пользуется готовым клиентом. В GitLab LDAP нет.

Группы участников и правила апрувов задаёт `idp.nemero.corp`.

## Вне скоупа

- Premium-функции: встроенные approval rules, Code Owners, group approval settings, compliance pipelines.
- CODEOWNERS и правила на пути файлов.
- SSO, Registry, Pages.
- Высокая доступность, GitLab Geo и установка GitLab Runner.
