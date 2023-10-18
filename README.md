### Описание Playbook

Playbook находится в файле [site.yml](./playbook/site.yml.)

Inventory находится в файле [prod.yml](./playbook/prod.yml.)

Переменные находятся в файле [vars.yml](./playbook/vars.yml.)

Playbook устанавливает сервисы clickhouse и vector.

Для установки сервиса clickhouse с сайта разработчика устанавливаются пакеты:

,,,
    clickhouse-common-static
    clickhouse-server
    clickhouse-client
,,,

После этого создаётся БД logs. Ввиду того, что доступ к СУБД появляется не сразу после запуска сервиса clickhouse-server, попытки создания БД выполняются в цикле, пока не увенчаются успехом.

В заключение в БД logs создаётся таблица logs, с полями, которые позволяют загружать записи из syslog-а:

,,,
CREATE TABLE IF NOT EXISTS logs.logs (
    apname String,
    facility String,
    hostname String,
    message String,
    msgid String,
    procid Int,
    severity String,
    timestamp String,
    version Int
) ENGINE = MergeTree ORDER BY timestamp;'
,,,

Для установки сервиса vector с сайта разработчика устанавливаются пакеты:

    vector

После из шаблона создаётся конфигурационный файл /etc/vector/vector.toml, который содержит набор команд для загрузки данных из syslog в БД clickhouse.

Playbook использует следующие переменные:

,,,
    clickhouse_version - требуемая версия clickhouse
    vector_version - требуемая версия vector
,,,
