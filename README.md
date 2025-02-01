# Домашнее задание к занятию "`Использование Ansible`" - `Дедюрин Денис`

## Подготовка к выполнению

1. Подготовьте в Yandex Cloud три хоста: для `clickhouse`, для `vector` и для `lighthouse`.
2. Репозиторий LightHouse находится [по ссылке](https://github.com/VKCOM/lighthouse).

## Основная часть

1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает LightHouse.
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать статику LightHouse, установить Nginx или любой другой веб-сервер, настроить его конфиг для открытия LightHouse, запустить веб-сервер.
4. Подготовьте свой inventory-файл `prod.yml`.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md-файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-03-yandex` на фиксирующий коммит, в ответ предоставьте ссылку на него.


### Ответ:

Дописал **playbook**
```
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
  tasks:
    - name: Install Clickhouse
      block:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
            mode: '0644'
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Get clickhouse distrib fallback
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"
            mode: '0644'
    - name: Install clickhouse packages
      become: true
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
        disable_gpg_check: true
        use_backend: "yum"
      notify: Start clickhouse service
    - name: Flush handlers
      ansible.builtin.meta: flush_handlers
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc != 82
      changed_when: create_db.rc == 0

- name: Install and configure Vector
  hosts: vector
  handlers:
    - name: Restart vector service
      become: true
      ansible.builtin.service:
        name: vector
        state: restarted
  tasks:
    - name: Install Vector
      block:
        - name: Download Vector package
          ansible.builtin.get_url:
            url: "https://packages.timber.io/vector/{{ vector_version }}/vector-{{ vector_version_suffix }}.x86_64.rpm"
            dest: "./vector-{{ vector_version }}.rpm"
            mode: '0644'
        - name: Install Vector
          become: true
          ansible.builtin.yum:
            name: "./vector-{{ vector_version }}.rpm"
            state: present
            disable_gpg_check: true
        - name: Deploy Vector config
          become: true
          ansible.builtin.template:
            src: "templates/vector.yaml.j2"
            dest: "/etc/vector/vector.yaml"
            mode: '0644'
          notify: Restart vector service

- name: Install Lighthouse
  hosts: lighthouse
  handlers:
    - name: Nginx reload
      become: true
      ansible.builtin.service:
        name: nginx
        state: restarted
  tasks:
    - name: Install dependencies
      block:
        - name: Install git
          become: true
          ansible.builtin.yum:
            name: git
            state: present
        - name: Install epel-release
          become: true
          ansible.builtin.yum:
            name: epel-release
            state: present
        - name: Install nginx
          become: true
          ansible.builtin.yum:
            name: nginx
            state: present
    - name: Configure nginx
      block:
        - name: Apply nginx config
          become: true
          ansible.builtin.template:
            src: nginx.conf.j2
            dest: /etc/nginx/nginx.conf
            mode: '0644'
    - name: Deploy Lighthouse
      block:
        - name: Clone Lighthouse repository
          become: true
          ansible.builtin.git:
            repo: "{{ lighthouse_url }}"
            dest: "{{ lighthouse_dir }}"
            version: master
        - name: Apply Lighthouse nginx config
          become: true
          ansible.builtin.template:
            src: lighthouse_nginx.conf.j2
            dest: /etc/nginx/conf.d/lighthouse.conf
            mode: "0644"
          notify: Nginx reload
```

Создал дополнительную группу, хосты и шаблоны.

Запустил команду:
```
ansible-playbook -i inventory/prod.yml site.yml --diff
```

