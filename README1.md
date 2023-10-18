# Домашнее задание к занятию 2 «Работа с Playbook» Баранов Сергей

## Подготовка к выполнению

1. * Необязательно. Изучите, что такое [ClickHouse](https://www.youtube.com/watch?v=fjTNS2zkeBs) и [Vector](https://www.youtube.com/watch?v=CgEhyffisLY).
2. Создайте свой публичный репозиторий на GitHub с произвольным именем или используйте старый.
3. Скачайте [Playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
4. Подготовьте хосты в соответствии с группами из предподготовленного playbook.

### Основная часть

1. Подготовьте свой inventory-файл `prod.yml`.

```
root@baranovsa:/home/baranovsa/8.2_Ansible_Playbook/playbook/inventory# cat prod.yml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 158.160.55.239
      ansible_connection: ssh
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
      ansible_python_interpreter: python3
root@baranovsa:/home/baranovsa/8.2_Ansible_Playbook/playbook/inventory#
```

2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev). Конфигурация vector должна деплоиться через template файл jinja2. От вас не требуется использовать все возможности шаблонизатора, просто вставьте стандартный конфиг в template файл. Информация по шаблонам по [ссылке](https://www.dmosk.ru/instruktions.php?object=ansible-nginx-install).

3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.

4. Tasks должны: скачать дистрибутив нужной версии, выполнить распаковку в выбранную директорию, установить vector.

site.yml
```
root@baranovsa:/home/baranovsa/8.2_Ansible_Playbook/playbook# cat site.yml
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      ansible.builtin.systemd:
        name: clickhouse-server
        state: started
        enabled: true
      become: true
      become_user: root
  tasks:
    - name: Install clickhouse amd64 packages from internet
      become: true
      become_user: root
      ansible.builtin.apt:
        deb: "https://packages.clickhouse.com/deb/pool/main/c/clickhouse-common-static/clickhouse-common-static_{{ clickhouse_version }}_amd64.deb"
      with_items: "{{ clickhouse_amd64_packages }}"
      notify: Start clickhouse service
    - name: Install clickhouse noarch packages from internet
      become: true
      become_user: root
      ansible.builtin.apt:
        deb: "https://packages.clickhouse.com/deb/pool/main/c/{{ item }}/{{ item }}_{{ clickhouse_version }}_all.deb"
      with_items: "{{ clickhouse_noarch_packages }}"
      notify: Start clickhouse service
    - name: Flush handlers
      ansible.builtin.meta: flush_handlers
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc != 82
      changed_when: create_db.rc == 0
      retries: 10
      delay: 3
      until: create_db.rc == 0 or create_db.rc == 82
    - name: Create table
      ansible.builtin.command: "clickhouse-client -q '
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
        ) ENGINE = MergeTree ORDER BY timestamp;'"
      register: create_table
      failed_when: create_table.rc != 0 and create_table.rc != 57
      changed_when: create_table.rc == 0
- name: Install Vector
  hosts: clickhouse
  handlers:
    - name: Restart vector service
      ansible.builtin.systemd:
        name: vector
        state: restarted
        enabled: true
      become: true
      become_user: root
  tasks:
    - name: Install vector packages from internet
      become: true
      become_user: root
      ansible.builtin.apt:
        deb: "https://packages.timber.io/vector/{{ vector_version }}/vector_{{ vector_version }}-1_amd64.deb"
      notify: Restart vector service
    - name: Template a config to /etc/vector/vector.toml
      become: true
      become_user: root
      ansible.builtin.template:
        src: templates/vector.toml
        dest: /etc/vector/vector.toml
        owner: root
        group: root
        mode: "0644"
      notify:
root@baranovsa:/home/baranovsa/8.2_Ansible_Playbook/playbook# 
```

5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.

```
root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-lint site.yml
root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook# 
```

6. Попробуйте запустить playbook на этом окружении с флагом `--check`.

```
root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i ./inventory/prod.yml site.yml --check

PLAY [Install Clickhouse] ****************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [clickhouse-01]

TASK [Install clickhouse amd64 packages from internet] ***********************************************
changed: [clickhouse-01] => (item=clickhouse-common-static)

TASK [Install clickhouse noarch packages from internet] **********************************************
failed: [clickhouse-01] (item=clickhouse-client) => {"ansible_loop_var": "item", "changed": false, "item": "clickhouse-client", "msg": "Dependency is not satisfiable: clickhouse-common-static (= 22.3.3.44)\n"}
failed: [clickhouse-01] (item=clickhouse-server) => {"ansible_loop_var": "item", "changed": false, "item": "clickhouse-server", "msg": "Dependency is not satisfiable: clickhouse-common-static (= 22.3.3.44)\n"}

RUNNING HANDLER [Start clickhouse service] ***********************************************************

PLAY RECAP *******************************************************************************************
clickhouse-01          	: ok=2	changed=1	unreachable=0	failed=1	skipped=0	rescued=0	ignored=0   

```

7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.

```
root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i ./inventory/prod.yml site.yml --diff

PLAY [Install Clickhouse] ****************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [clickhouse-01]

TASK [Install clickhouse amd64 packages from internet] ***********************************************
ok: [clickhouse-01] => (item=clickhouse-common-static)

TASK [Install clickhouse noarch packages from internet] **********************************************
ok: [clickhouse-01] => (item=clickhouse-client)
ok: [clickhouse-01] => (item=clickhouse-server)

TASK [Create database] *******************************************************************************
ok: [clickhouse-01]

TASK [Create table] **********************************************************************************
changed: [clickhouse-01]

PLAY [Install Vector] ********************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [clickhouse-01]

TASK [Install vector packages from internet] *********************************************************
Selecting previously unselected package vector.
(Reading database ... 109792 files and directories currently installed.)
Preparing to unpack .../vector_0.21.1-1_amd64f8ep93_v.deb ...
Unpacking vector (0.21.1-1) ...
Setting up vector (0.21.1-1) ...
systemd-journal:x:101:
changed: [clickhouse-01]

TASK [Template a config to /etc/vector/vector.toml] **************************************************
--- before: /etc/vector/vector.toml
+++ after: /root/.ansible/tmp/ansible-local-4587kmurft4o/tmp7qb45rq0/vector.toml
@@ -1,44 +1,38 @@
-#                                	__   __  __
-#                                	\ \ / / / /
-#                                 	\ V / / /
-#                                  	\_/  \/
-#
-#                                	V E C T O R
-#                               	Configuration
-#
-# ------------------------------------------------------------------------------
-# Website: https://vector.dev
-# Docs: https://vector.dev/docs
-# Chat: https://chat.vector.dev
-# ------------------------------------------------------------------------------
-
-# Change this to use a non-default directory for Vector data storage:
-# data_dir = "/var/lib/vector"
-
 # Random Syslog-formatted logs
-[sources.dummy_logs]
-type = "demo_logs"
-format = "syslog"
-interval = 1
+[sources.syslog]
+type = "file"
+#type = "demo_logs"
+include = [ "/var/log/syslog" ]
+read_from = "end"
+#format = "syslog"
+#interval = 1
 
 # Parse Syslog logs
 # See the Vector Remap Language reference for more info: https://vrl.dev
-[transforms.parse_logs]
+[transforms.parse_syslog]
 type = "remap"
-inputs = ["dummy_logs"]
+inputs = ["syslog"]
 source = '''
 . = parse_syslog!(string!(.message))
 '''
 
 # Print parsed logs to stdout
-[sinks.print]
-type = "console"
-inputs = ["parse_logs"]
-encoding.codec = "json"
+#[sinks.print]
+#type = "console"
+#inputs = ["parse_logs"]
+#encoding.codec = "json"
 
 # Vector's GraphQL API (disabled by default)
 # Uncomment to try it out with the `vector top` command or
 # in your browser at http://localhost:8686
-#[api]
-#enabled = true
-#address = "127.0.0.1:8686"
+[api]
+enabled = true
+address = "0.0.0.0:8686"
+
+[sinks.clickhouse]
+type = "clickhouse"
+inputs = ["syslog"]
+endpoint = "http://localhost:8123"
+database = "logs"
+table = "logs"
+skip_unknown_fields = true

changed: [clickhouse-01]

RUNNING HANDLER [Restart vector service] *************************************************************
changed: [clickhouse-01]

PLAY RECAP *******************************************************************************************
clickhouse-01          	: ok=9	changed=4	unreachable=0	failed=0	skipped=0	rescued=0	ignored=0   

root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook#
```

8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.

```
root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i ./inventory/prod.yml site.yml --diff

PLAY [Install Clickhouse] ****************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [clickhouse-01]

TASK [Install clickhouse amd64 packages from internet] ***********************************************
ok: [clickhouse-01] => (item=clickhouse-common-static)

TASK [Install clickhouse noarch packages from internet] **********************************************
ok: [clickhouse-01] => (item=clickhouse-client)
ok: [clickhouse-01] => (item=clickhouse-server)

TASK [Create database] *******************************************************************************
ok: [clickhouse-01]

TASK [Create table] **********************************************************************************
changed: [clickhouse-01]

PLAY [Install Vector] ********************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [clickhouse-01]

TASK [Install vector packages from internet] *********************************************************
ok: [clickhouse-01]

TASK [Template a config to /etc/vector/vector.toml] **************************************************
ok: [clickhouse-01]

PLAY RECAP *******************************************************************************************
clickhouse-01          	: ok=8	changed=1	unreachable=0	failed=0	skipped=0	rescued=0	ignored=0   

root@baranovsa:/home/baranovsa/mnt-homeworks/08-ansible-02-playbook/playbook#
```

9. Подготовьте README.md-файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги. Пример качественной документации ansible playbook по [ссылке](https://github.com/opensearch-project/ansible-playbook).

10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.

---

