# Апрувы на idp.nemero.corp

GitLab CE не умеет обязательные approval rules. На ВМ GitLab остаётся только hook. Группы участников, хранение правил и решение «можно ли влить» делает сервис на `idp.nemero.corp`.

Пока сервиса нет, в `ansible/inventories/example/group_vars/gitlab/vars.yml` стоит `gitlab_approval_check_enabled: false`. Hook записан, но наружу не ходит и push не блокирует. Когда сервис отвечает, флаг ставят в `true` и прогоняют плейбук.

## Что уже делает GitLab

При включённом флаге hook на каждую запись в репозиторий отправляет `POST` на `https://idp.nemero.corp/approvals/check`.

Тело:

```json
{
  "project_path": "payments/api",
  "username": "a.nazarov",
  "updates": [
    {
      "oldrev": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "newrev": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "ref": "refs/heads/main",
      "parents": ["..."],
      "message": "Merge branch 'feature' into 'main'\n\nSee merge request payments/api!12\n",
      "patch_id": "..."
    }
  ]
}
```

Ответ `200` и JSON:

```json
{"allow": true, "message": ""}
```

или

```json
{"allow": false, "message": "Отклонено политикой апрувов.\nНе выполнено:\n- security: 0/1, нужна группа security"}
```

`allow: true` пропускает запись. Любой другой ответ, обрыв, таймаут дольше 20 секунд или не-JSON отклоняют её. Текст `message` человек видит при Merge.

Кнопка Approve в GitLab штатная. Кнопка Merge остаётся нажатой: запрет срабатывает в момент записи в ветку. Владелец проекта hook не выключает.

Шаблон сообщения merge-коммита уже задан плейбуком и содержит строку `See merge request %{reference}`.

## Группы участников

Состав групп хранит `idp.nemero.corp`. В GitLab эти группы не создаются.

Источник людей — Keycloak, realm `gitlab`, issuer `https://sso.nemero.online/realms/gitlab`. Людей и группы в Keycloak кладёт другой плейбук (федерация с LDAP). Обратно в каталог сервис не пишет.

Имя группы в правиле — это имя группы у сервиса, то же, что в Keycloak: `security`, `devops`, `docs`.

- Состав берётся из Keycloak и лежит в хранилище сервиса.
- Для кнопки Approve человек по-прежнему должен быть участником проекта в GitLab с ролью не ниже Developer. Список тех, чей апрув засчитывается, сервис берёт из своего хранилища, не из групп GitLab.
- Локальных пользователей `root`, `ghost`, `support-bot`, `alert-bot` сервис не добавляет в свои группы.
- Группу `a_gitlab_admins` этот сервис не трогает. Её обрабатывает вход SSO на самом GitLab: участник становится администратором инстанса.

Сервису нужен токен GitLab, чтобы читать merge request и апрувы. Менять группы GitLab им не нужно. Токен хранится на `idp.nemero.corp`, не на ВМ GitLab.

## Правила апрувов

Правила хранит и меняет только `idp.nemero.corp`. Копия в GitLab ничего не запрещает. Оператор правит правила там: файл на сервере idp или его API. После сохранения следующая проверка hook уже видит новый файл. Отдельный прогон плейбука GitLab для смены правила не нужен.

Формат:

```yaml
version: 1

defaults:
  prevent_author_approval: true
  rules: []

rulesets:
  platform:
    rules:
      - name: security
        approver_group: security
        approvals_required: 1
      - name: devops
        approver_group: devops
        approvals_required: 1
  docs:
    rules:
      - name: maintainers
        approver_group: docs
        approvals_required: 1

bindings:
  - match: { group: payments }
    ruleset: platform
  - match: { group: payments/sandbox }
    ruleset: docs
  - match: { project: payments/legacy-readme }
    rules: []
```

Как выбирать набор правил для `project_path`:

- Совпадает самый длинный путь.
- Точный проект важнее группы той же длины.
- Нет совпадения — берутся `defaults`.
- `rules: []` значит обязательных апрувов нет, запись в защищённую ветку через merge request разрешена.
- Неизвестный `ruleset` или `approvals_required` меньше 1 — отказ, не пропуск.
- Файл не читается — отказ.

Правила одного набора складываются по И. `approvals_required` — сколько разных людей из этой группы должны апрувнуть. Один человек закрывает не больше одного правила. Если он входит в несколько групп, его апрув забирает то правило, где подходящих людей меньше. Апрув автора не считается, когда `prevent_author_approval: true`. Апрув бота не считается. Апрув, поставленный раньше текущего коммита MR, не считается.

Пустая группа апруверов правило не закрывает.

Пример выше читается так:

- проекты внутри `payments` требуют один апрув security и один отдельный апрув devops;
- `payments/sandbox` и всё внутри требуют только docs, потому что путь длиннее;
- проект `payments/legacy-readme` обязательных апрувов не требует.

Рабочая копия правил лежит только на `idp.nemero.corp`.

## Как решать /approvals/check

Для каждого элемента `updates`:

1. Это не ветка (`refs/heads/`), удаление ветки или первый коммит, который ветку создаёт — пропустить.
2. Ветка не защищена — пропустить. Защищённые имена сейчас `main` и `master`, плюс то, что GitLab отдаёт в protected branches проекта. Ошибка чтения protected branches — отказ.
3. Правил для проекта нет — пропустить.
4. Иначе найти открытый merge request. Сначала строка `See merge request <проект>!<номер>` в сообщении коммита, путь должен совпасть с `project_path`. Потом SHA MR, затем родитель коммита, затем один совпавший `patch_id`. MR нет — отказ: в защищённую ветку можно только влить merge request.
5. Ветка MR должна совпадать с веткой записи.
6. Прочитать `GET /projects/:id/merge_requests/:iid/approvals`. Кто входит в `approver_group`, сервис смотрит в своём хранилище.
7. Все правила закрыты — пропуск. Иначе отказ и текст, какое правило не закрыто и какая группа нужна.

Ошибка GitLab API на любом из этих шагов — отказ.

Чеклист в обсуждении MR сервис может писать своим токеном: один комментарий с маркером `<!-- gitlab-approval-policy -->` и дальше править тот же комментарий. На запрет влития это не влияет. Запрещает только ответ hook.

## Что удалить в GitLab сейчас

Это остатки локального сервиса. На апрувы они уже не влияют.

В интерфейсе:

- Пользователь `approval-bot`. Вместе с ним исчезнет его токен.
- Проект `approvers/approval-policy`. Это была копия правил, она ничего не запрещает.
- Группу `approvers` вместе с `approvers/devops` и остальными подгруппами. Состав этих групп внешний сервис хранит у себя.
- Группу `ops`, если она пустая. Она осталась от старого пути проекта политики.
- System hook с адресом `http://127.0.0.1:8765/webhook` (Admin → System hooks).

Локального `root` не удалять.

На ВМ GitLab:

```bash
systemctl disable --now gitlab-approval access-sync.timer access-sync.service
rm -f /etc/systemd/system/gitlab-approval.service \
  /etc/systemd/system/access-sync.service \
  /etc/systemd/system/access-sync.timer
systemctl daemon-reload
rm -rf /opt/gitlab-approval /etc/gitlab-approval /var/lib/gitlab-approval
```

Файл `/var/opt/gitlab/gitaly/custom_hooks/pre-receive.d/approval-policy` не удалять: это hook-заглушка. Повторный прогон плейбука снимает те же unit и каталоги и выключает system hook на `127.0.0.1:8765`, если токен бота ещё лежит на диске.

## Чего сервис не делает

- Не заменяет экран Approval rules из Premium. В интерфейсе GitLab остаётся обычная кнопка Approve.
- Не выключает пароль, SSO и группу `a_gitlab_admins`. Это настройки ВМ GitLab.
- Не слушает на самой ВМ GitLab и не кладёт туда файл правил.
