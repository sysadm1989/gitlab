# CrowdSec

CrowdSec читает логи SSH и nginx GitLab и просит файрвол временно закрыть адрес, с которого идёт перебор или типовая атака. Это фильтр на nftables, не модуль внутри nginx GitLab.

Команды ниже выполняются на ВМ от root, через `sudo`.

## Что он смотрит

- Неудачные входы по SSH.
- Типовые HTTP-атаки и известные уязвимости в логе `/var/log/gitlab/nginx/gitlab_access.log`.
- Подбор пароля на страницах входа GitLab: `POST /users/sign_in`, `/users/password`, `/oauth/token`. Форма входа отвечает кодом 200, поэтому для неё отдельное правило: больше 5 таких запросов подряд с одного адреса.

Общий список плохих адресов CrowdSec подтягивается сам. Личный кабинет для этого не нужен.

## Белый список

Адреса из белого списка не банятся. Туда уже входят localhost и сети из `crowdsec_whitelist_cidrs` в `ansible/inventories/example/group_vars/gitlab/vars.yml`. Сейчас там `10.0.0.0/8`.

Чтобы добавить сеть, допишите её в этот список и прогоните плейбук:

```yaml
crowdsec_whitelist_cidrs:
  - 10.0.0.0/8
  - 203.0.113.0/24
```

Правка файла на сервере при следующем прогоне затрётся. Источник один: `vars.yml`.

## Посмотреть, кто забанен

```bash
sudo cscli decisions list
```

В колонке `expiration` видно, когда бан спадёт сам. Пустой список значит, что сейчас никто не заблокирован.

Почему адрес попал в бан:

```bash
sudo cscli alerts list
sudo cscli alerts inspect <номер>
```

Номер берётся из первой колонки `alerts list`.

## Снять бан

Один адрес:

```bash
sudo cscli decisions delete --ip 203.0.113.10
```

Все баны сразу:

```bash
sudo cscli decisions delete --all
```

Снятие бана не добавляет адрес в белый список. Если поведение повторится, бан вернётся. Постоянное исключение делается через `crowdsec_whitelist_cidrs` и повторный прогон плейбука.

## Проверить, что фильтр жив

```bash
sudo systemctl status crowdsec crowdsec-firewall-bouncer
sudo cscli metrics
```

Оба сервиса должны быть `active`. Если `crowdsec` работает, а `crowdsec-firewall-bouncer` нет, решения есть, но файрвол их не применяет.

Последние сообщения движка:

```bash
sudo journalctl -u crowdsec -n 50 --no-pager
```

## Если забанило вас

Зайдите на ВМ с адреса из `10.0.0.0/8` или через консоль гипервизора и снимите бан командой `cscli decisions delete --ip`. После этого добавьте свою сеть в `crowdsec_whitelist_cidrs`, если её там ещё нет.

## Консоль CrowdSec

Необязательна. Пустой `crowdsec_enroll_key` в `vars.yml` означает работу без личного кабинета: локальные баны и общий список адресов всё равно действуют.

Ключ с [app.crowdsec.net](https://app.crowdsec.net) кладётся в `crowdsec_enroll_key`. Плейбук регистрирует машину один раз. Повторный прогон регистрацию не повторяет.
