# САНКТ–ПЕТЕРБУРГСКИЙ НАЦИОНАЛЬНЫЙ ИССЛЕДОВАТЕЛЬСКИЙ УНИВЕРСИТЕТ ИНФОРМАЦИОННЫХ ТЕХНОЛОГИЙ, МЕХАНИКИ И ОПТИКИ
## ФАКУЛЬТЕТ ИНФОКОММУНИКАЦИОННЫХ ТЕХНОЛОГИЙ

**Отчёт по лабораторной работе 3**

по курсу «Администрирование компьютерных сетей»

**Ansible + Caddy**

**Выполнили:** Кутуков.Д, Кижваткин Н, Фоченков С, Бочарников М, Субботин

**Проверил:** Самохин Н.Ю.

Санкт–Петербург 2025 г.

---

## Содержание

1. [Введение](#введение)
2. [Часть 1. Продвинутая установка и настройка Ansible](#часть-1-продвинутая-установка-и-настройка-ansible)
   - [1.1 Установка Ansible с дополнительными зависимостями](#11-установка-ansible-с-дополнительными-зависимостями)
   - [1.2 Настройка динамического inventory](#12-настройка-динамического-inventory)
   - [1.3 Использование фактов для проверки подключения](#13-использование-фактов-для-проверки-подключения)
3. [Часть 2. Модульная установка Caddy с использованием подролей](#часть-2-модульная-установка-caddy-с-использованием-подролей)
   - [2.1 Создание иерархии ролей и подролей](#21-создание-иерархии-ролей-и-подролей)
   - [2.2 Реализация модульных задач с обработчиками](#22-реализация-модульных-задач-с-обработчиками)
   - [2.3 Интеграция всех компонентов через основной playbook](#23-интеграция-всех-компонентов-через-основной-playbook)
4. [Часть 3. Продвинутые шаблоны и управление конфигурацией](#часть-3-продвинутые-шаблоны-и-управление-конфигурацией)
   - [3.1 Динамическая регистрация доменов через переменные](#31-динамическая-регистрация-доменов-через-переменные)
   - [3.2 Условная генерация конфигурации с использованием фильтров Jinja2](#32-условная-генерация-конфигурации-с-использованием-фильтров-jinja2)
   - [3.3 Валидация конфигурации и тестирование перед применением](#33-валидация-конфигурации-и-тестирование-перед-применением)
5. [Задания](#задания)
   - [5.1 Создание переиспользуемых playbook'ов для операций с файлами](#51-создание-переиспользуемых-playbook'ов-для-операций-с-файлами)
   - [5.2 Расширенная конфигурация с использованием переменных и фактов](#52-расширенная-конфигурация-с-использованием-переменных-и-фактов)
6. [Выводы](#выводы)
7. [Приложение A: Полная структура проекта](#приложение-a-полная-структура-проекта)
8. [Приложение B: Примеры конфигурационных файлов](#приложение-b-примеры-конфигурационных-файлов)

---

## Введение

**Цель работы** освоить продвинутые возможности Ansible, включая динамический inventory, модульную архитектуру с подролями, обработчики (handlers), условное выполнение, фильтры Jinja2 и валидацию конфигурации. Автоматизировать установку и настройку веб-сервера Caddy используя децентрализованный подход, где каждый компонент системы управляется отдельным модулем.

### Задание

1. Установить Ansible с расширенными зависимостями (molecule для тестирования).
2. Создать динамический inventory на основе JSON/YAML.
3. Использовать фильтры Jinja2 и условное выполнение для гибкой конфигурации.
4. Реализовать иерархию ролей и подролей (dependencies).
5. Использовать обработчики (handlers) для управления жизненным циклом сервиса.
6. Настроить валидацию конфигурации перед применением.
7. Создать переиспользуемые playbook'ы с параметризацией.
8. Расширить функциональность Caddy через множественные переменные.

### Описание используемого окружения

Лабораторная работа выполнялась в среде Linux с использованием контролёра Ansible и локального подключения. Для тестирования конфигураций использовалась утилита **Molecule**, которая позволяет тестировать роли Ansible в изолированной среде.

**Используемые компоненты:**

- Операционная система: Ubuntu 22.04 LTS
- Python версии 3.10+
- Ansible версии 2.14+
- Molecule для тестирования ролей
- Caddy веб-сервер
- yamllint для валидации YAML файлов
- Ansible Lint для проверки качества playbook'ов

---

## Часть 1. Продвинутая установка и настройка Ansible

### 1.1 Установка Ansible с дополнительными зависимостями

На первом этапе была выполнена установка Ansible вместе с дополнительными инструментами для разработки, тестирования и валидации:

**Листинг 1: Установка Ansible и инструментов**

```bash
pip install ansible==2.14.0
pip install molecule==6.0.0
pip install ansible-lint==6.20.0
pip install yamllint==2.32.0
pip install jinja2==3.1.2
pip install netaddr==0.9.0  # для работы с IP-адресами
```

**Листинг 2: Проверка версии Ansible**

```bash
ansible --version
ansible [core 2.14.0]
  config file = /path/to/ansible.cfg
  configured module search path = ['/path/to/roles']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.10.12
```

После установки был создан расширенный файл **ansible.cfg** с дополнительными параметрами:

**Листинг 3: ansible.cfg (продвинутая конфигурация)**

```ini
[defaults]
# Инвентари и фильтры
inventory = inventory/
host_key_checking = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_cache
fact_caching_timeout = 3600

# Вывод и отладка
stdout_callback = yaml
callback_whitelist = profile_tasks
force_color = True
display_skipped_hosts = False

# Параметры выполнения
timeout = 10
roles_path = roles:galaxy-roles
retry_files_enabled = True
retry_files_save_path = /tmp

# Опции privilige escalation
[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

### 1.2 Настройка динамического inventory

Вместо статического файла hosts было создано три типа inventory:

**Листинг 4: inventory/static_hosts.yml**

```yaml
---
ungrouped: {}

all:
  vars:
    ansible_connection: local
    ansible_python_interpreter: /usr/bin/python3
    
  children:
    web_servers:
      vars:
        server_role: web
        enable_ssl: true
      hosts:
        localhost:
          ansible_host: 127.0.0.1
          ansible_port: 22
          server_name: "web-server-01"
          
    caddy_nodes:
      vars:
        caddy_version: latest
      hosts:
        localhost:
          ansible_host: 127.0.0.1
```

**Листинг 5: inventory/dynamic_inventory.py (скрипт для динамического inventory)**

```python
#!/usr/bin/env python3
import json
import sys
import socket

def get_dynamic_inventory():
    """Генерирует динамический inventory на основе системных параметров"""
    inventory = {
        "all": {
            "hosts": {
                "localhost": {
                    "ansible_connection": "local",
                    "hostname": socket.gethostname(),
                    "server_role": "web",
                    "environment": "production"
                }
            },
            "vars": {
                "ansible_python_interpreter": "/usr/bin/python3"
            }
        },
        "_meta": {
            "hostvars": {
                "localhost": {
                    "caddy_port": 80,
                    "caddy_secure_port": 443,
                    "enable_ssl": True,
                    "cache_dir": "/var/cache/caddy"
                }
            }
        }
    }
    return inventory

if __name__ == "__main__":
    if len(sys.argv) == 2 and sys.argv[1] == "--list":
        print(json.dumps(get_dynamic_inventory(), indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == "--host":
        print(json.dumps(get_dynamic_inventory()["_meta"]["hostvars"].get(sys.argv[2], {}), indent=2))
    else:
        print("Usage: dynamic_inventory.py --list | --host <hostname>")
        sys.exit(1)
```

**Листинг 6: Тестирование динамического inventory**

```bash
chmod +x inventory/dynamic_inventory.py
./inventory/dynamic_inventory.py --list | head -20
```

**Результат:**

```json
{
  "all": {
    "hosts": {
      "localhost": {
        "ansible_connection": "local",
        "hostname": "ubuntu-22",
        "server_role": "web",
        "environment": "production"
      }
    },
    ...
  }
}
```

### 1.3 Использование фактов для проверки подключения

Вместо простой команды ping была использована система фактов Ansible для получения подробной информации о целевом хосте:

**Листинг 7: Использование фактов для проверки**

```bash
ansible all -i inventory/ -m setup -a "filter=ansible_os_family,ansible_distribution*"
```

**Результат:**

```
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "Ubuntu",
        "ansible_distribution_major_version": "22",
        "ansible_distribution_release": "jammy",
        "ansible_distribution_version": "22.04",
        "ansible_os_family": "Debian"
    }
}
```

Был создан специальный playbook для валидации окружения:

**Листинг 8: playbooks/validate_environment.yml**

```yaml
---
- name: Validate Target Environment
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Check Python version
      assert:
        that:
          - ansible_python_version is version('3.8', '>=')
        fail_msg: "Python 3.8+ required"
      
    - name: Check OS Family
      assert:
        that:
          - ansible_os_family in ['Debian', 'RedHat']
        fail_msg: "Only Debian/RedHat based systems supported"
    
    - name: Display system information
      debug:
        msg: |
          System: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Kernel: {{ ansible_kernel }}
          CPU cores: {{ ansible_processor_vcpus }}
          Free Memory: {{ (ansible_memfree_mb / 1024) | round(2) }} GB

    - name: Check if Caddy already installed
      stat:
        path: /usr/bin/caddy
      register: caddy_check
    
    - name: Report Caddy status
      debug:
        msg: "Caddy is {{ 'already installed' if caddy_check.stat.exists else 'not installed' }}"
```

**Листинг 9: Запуск валидации**

```bash
ansible-playbook playbooks/validate_environment.yml -i inventory/
```

---

## Часть 2. Модульная установка Caddy с использованием подролей

### 2.1 Создание иерархии ролей и подролей

Вместо одной монолитной роли была создана иерархия ролей с чётким разделением ответственности:

**Листинг 10: Структура ролей**

```
roles/
├── caddy/
│   ├── meta/
│   │   └── main.yml           # Зависимости роли
│   ├── defaults/
│   │   └── main.yml           # Переменные по умолчанию
│   ├── vars/
│   │   └── main.yml           # Внутренние переменные
│   ├── tasks/
│   │   ├── main.yml           # Основные задачи
│   │   └── validation.yml     # Валидация перед установкой
│   ├── handlers/
│   │   └── main.yml           # Обработчики событий
│   ├── templates/
│   │   ├── caddyfile.j2
│   │   ├── index.html.j2
│   │   └── caddy.service.j2
│   └── files/
│       └── caddy_check.sh
│
├── caddy-dependencies/
│   ├── tasks/
│   │   ├── main.yml
│   │   └── apt-setup.yml
│   └── files/
│       └── caddy-sources.txt
│
├── caddy-config/
│   ├── tasks/
│   │   ├── main.yml
│   │   ├── ssl-setup.yml
│   │   └── headers-setup.yml
│   └── templates/
│       └── caddyfile-extensions.j2
│
└── caddy-service/
    ├── tasks/
    │   └── main.yml
    └── handlers/
        └── main.yml
```

**Листинг 11: roles/caddy/meta/main.yml (определение зависимостей)**

```yaml
---
galaxy_info:
  author: 'Lab Team'
  license: 'MIT'
  min_ansible_version: 2.10
  galaxy_tags:
    - webserver
    - caddy

dependencies:
  - role: caddy-dependencies
    vars:
      install_method: apt
  
  - role: caddy-config
    vars:
      config_priority: high
```

**Листинг 12: roles/caddy/defaults/main.yml**

```yaml
---
# Основные переменные Caddy
caddy_version: "latest"
caddy_user: "caddy"
caddy_group: "caddy"
caddy_home: "/var/lib/caddy"
caddy_config_dir: "/etc/caddy"

# Параметры конфигурации
caddy_domains: []
caddy_email: "admin@example.com"
caddy_enable_ssl: true
caddy_enable_http2: true
caddy_enable_gzip: true

# Параметры сервиса
caddy_service_enabled: true
caddy_service_state: "started"
caddy_service_restart_on_change: true

# Логирование
caddy_log_level: "info"
caddy_access_log_enabled: true
caddy_access_log_path: "/var/log/caddy/access.log"

# Безопасность
caddy_security_headers: true
caddy_custom_headers: {}

# Валидация
caddy_validate_config: true
caddy_test_config_before_reload: true
```

### 2.2 Реализация модульных задач с обработчиками

**Листинг 13: roles/caddy-dependencies/tasks/main.yml**

```yaml
---
- name: Include OS-specific setup
  include_tasks: "apt-setup.yml"
  when: ansible_os_family == "Debian"

- name: Update package cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Install system dependencies
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - gnupg
    - apt-transport-https
    - ca-certificates
    - lsb-release
  when: ansible_os_family == "Debian"
```

**Листинг 14: roles/caddy-dependencies/tasks/apt-setup.yml**

```yaml
---
- name: Add Caddy GPG key
  apt_key:
    url: "https://dl.caddy.community/linux/caddy.gpg"
    state: present
    keyring: "/usr/share/keyrings/caddy-archive-keyring.gpg"

- name: Add Caddy repository
  apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/caddy-archive-keyring.gpg] https://dl.caddy.community/linux/debian any main"
    state: present
    filename: caddy-stable

- name: Install Caddy package
  apt:
    name: "caddy"
    state: "{{ 'latest' if caddy_version == 'latest' else caddy_version }}"
    update_cache: yes
  notify: "restart caddy"
```

**Листинг 15: roles/caddy/tasks/main.yml (основные задачи)**

```yaml
---
- name: Validate environment
  include_tasks: validation.yml

- name: Create Caddy directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ caddy_user }}"
    group: "{{ caddy_group }}"
    mode: "0750"
  loop:
    - "{{ caddy_home }}"
    - "{{ caddy_config_dir }}"
    - "/var/log/caddy"

- name: Generate Caddyfile from template
  template:
    src: caddyfile.j2
    dest: "{{ caddy_config_dir }}/Caddyfile"
    owner: "{{ caddy_user }}"
    group: "{{ caddy_group }}"
    mode: "0640"
    validate: "caddy validate --config %s"
  notify: 
    - test caddy config
    - reload caddy

- name: Create web root directory
  file:
    path: "/var/www/html"
    state: directory
    owner: "{{ caddy_user }}"
    group: "{{ caddy_group }}"
    mode: "0755"

- name: Generate index.html from template
  template:
    src: index.html.j2
    dest: "/var/www/html/index.html"
    owner: "{{ caddy_user }}"
    group: "{{ caddy_group }}"
    mode: "0644"

- name: Ensure Caddy service is started and enabled
  systemd:
    name: caddy
    state: "{{ caddy_service_state }}"
    enabled: "{{ caddy_service_enabled }}"
    daemon_reload: true
  when: caddy_service_restart_on_change
```

**Листинг 16: roles/caddy/tasks/validation.yml**

```yaml
---
- name: Assert required variables are defined
  assert:
    that:
      - caddy_config_dir is defined
      - caddy_user is defined
      - caddy_domains is defined
    fail_msg: "Required Caddy variables are not defined"

- name: Check if Caddy binary exists
  stat:
    path: "/usr/bin/caddy"
  register: caddy_binary
  failed_when: not caddy_binary.stat.exists
  ignore_errors: true

- name: Display validation results
  debug:
    msg: |
      Validation Summary:
      - Caddy binary: {{ 'Found' if caddy_binary.stat.exists else 'Not found' }}
      - Config directory: {{ caddy_config_dir }}
      - Service user: {{ caddy_user }}
      - Domains configured: {{ caddy_domains | length }}
```

**Листинг 17: roles/caddy/handlers/main.yml**

```yaml
---
- name: test caddy config
  shell: |
    /usr/bin/caddy validate --config {{ caddy_config_dir }}/Caddyfile
  become: true
  listen: "reload caddy"

- name: reload caddy
  systemd:
    name: caddy
    state: reloaded
  become: true
  listen: "reload caddy"

- name: restart caddy
  systemd:
    name: caddy
    state: restarted
  become: true

- name: caddy config changed
  debug:
    msg: "Caddy configuration has been updated"
```

### 2.3 Интеграция всех компонентов через основной playbook

**Листинг 18: playbooks/deploy_caddy.yml**

```yaml
---
- name: Deploy Caddy Web Server with Advanced Configuration
  hosts: caddy_nodes
  become: true
  gather_facts: true
  
  pre_tasks:
    - name: Validate deployment prerequisites
      block:
        - name: Run environment validation
          include_role:
            name: caddy
            tasks_from: validation
        
        - name: Collect facts
          setup:
            gather_subset: 
              - '!all'
              - 'distribution'
              - 'os'
              - 'memory'

    - name: Display deployment info
      debug:
        msg: |
          Deployment Information:
          - Target Host: {{ inventory_hostname }}
          - OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          - Caddy Version: {{ caddy_version }}
          - Domains: {{ caddy_domains | join(', ') }}
  
  roles:
    - role: caddy
      vars:
        caddy_domains: "{{ groups['caddy_nodes'] | map(attribute='caddy_domains', default=[]) | list }}"
        caddy_enable_ssl: "{{ enable_ssl | default(true) }}"
  
  post_tasks:
    - name: Verify Caddy is running
      systemd:
        name: caddy
        state: started
      register: caddy_status
    
    - name: Check Caddy ports
      wait_for:
        host: "{{ ansible_default_ipv4.address }}"
        port: "{{ item }}"
        timeout: 5
      loop:
        - 80
        - 443
      ignore_errors: true
    
    - name: Validate web server response
      uri:
        url: "http://localhost"
        status_code: 200
      register: web_response
      ignore_errors: true
    
    - name: Display final status
      debug:
        msg: |
          Deployment Complete!
          - Caddy Status: {{ caddy_status.status.ActiveState }}
          - HTTP Response: {{ web_response.status | default('N/A') }}
```

---

## Часть 3. Продвинутые шаблоны и управление конфигурацией

### 3.1 Динамическая регистрация доменов через переменные

Вместо статического списка доменов используется гибкая система с использованием group_vars и host_vars:

**Листинг 19: group_vars/caddy_nodes.yml**

```yaml
---
# Список доменов с расширенной конфигурацией
caddy_domains_config:
  - domain: "example.com"
    aliases:
      - "www.example.com"
      - "api.example.com"
    email: "admin@example.com"
    enable_ssl: true
    enable_http_redirect: true
    security_headers:
      - name: "X-Frame-Options"
        value: "SAMEORIGIN"
      - name: "X-Content-Type-Options"
        value: "nosniff"
      - name: "Strict-Transport-Security"
        value: "max-age=31536000; includeSubDomains"
  
  - domain: "localhost"
    aliases: []
    email: "local@localhost"
    enable_ssl: false
    enable_http_redirect: false
    security_headers: []

# Переменные по умолчанию для всех доменов
caddy_global_config:
  log_level: "info"
  auto_https: "on"
  grace_period: "5s"
  
# Расширенная фильтрация
caddy_enabled_domains: "{{ caddy_domains_config | selectattr('enable_ssl', 'equalto', true) | map(attribute='domain') | list }}"
```

**Листинг 20: Использование динамических переменных в playbook**

```yaml
---
- name: Deploy domains dynamically
  hosts: caddy_nodes
  gather_facts: true
  
  tasks:
    - name: Display enabled domains
      debug:
        msg: "Deploying to domains: {{ caddy_enabled_domains | join(', ') }}"
    
    - name: Create domain-specific configurations
      include_tasks: domain_setup.yml
      vars:
        target_domain: "{{ item }}"
      loop: "{{ caddy_domains_config }}"
      loop_control:
        label: "{{ item.domain }}"
```

### 3.2 Условная генерация конфигурации с использованием фильтров Jinja2

**Листинг 21: roles/caddy/templates/caddyfile.j2 (продвинутый шаблон)**

```jinja2
# Caddyfile - автогенерировано из Ansible
# Сгенерировано: {{ ansible_date_time.iso8601 }}
# Хост: {{ inventory_hostname }}

{%- if caddy_global_config is defined %}
{
  log_level {{ caddy_global_config.log_level | default('info') }}
  auto_https {{ caddy_global_config.auto_https | default('on') }}
  grace_period {{ caddy_global_config.grace_period | default('5s') }}
}
{%- endif %}

{%- for domain_config in caddy_domains_config %}

# Конфигурация для домена: {{ domain_config.domain }}
{{ domain_config.domain }} {
  {%- if domain_config.aliases %}
  # Aliases
  {%- for alias in domain_config.aliases %}
  alias {{ alias }}
  {%- endfor %}
  {%- endif %}

  {%- if domain_config.enable_ssl %}
  # Автоматический TLS от Let's Encrypt
  tls {{ domain_config.email }}
  {%- else %}
  # Внутренняя конфигурация (без TLS)
  {%- endif %}

  {%- if domain_config.enable_http_redirect and domain_config.enable_ssl %}
  # Редирект HTTP на HTTPS
  redir http:// https:// permanent
  {%- endif %}

  # Корневая директория
  root * /var/www/html

  # Обслуживание файлов
  file_server {
    index index.html
    hide .env .git .htaccess
  }

  {%- if domain_config.security_headers %}
  # Заголовки безопасности
  header {
    {%- for header in domain_config.security_headers %}
    {{ header.name }} "{{ header.value }}"
    {%- endfor %}
    X-UA-Compatible "IE=edge"
    Referrer-Policy "strict-origin-when-cross-origin"
  }
  {%- endif %}

  {%- if caddy_enable_gzip %}
  # Сжатие
  encode gzip
  {%- endif %}

  # Логирование
  log {
    output file {{ caddy_access_log_path }} {
      roll_size 100mb
      roll_keep 3
      roll_keep_for 7d
    }
    format json
  }

  # Обработчик ошибок
  handle_errors {
    respond "{http.error.status_code} {http.error.status_text}" {http.error.status_code}
  }
}

{%- endfor %}

{%- if caddy_enable_health_check %}
# Health check endpoint
:{{ caddy_health_check_port | default(8080) }} {
  respond /health 200
  respond /ready 200
}
{%- endif %}
```

**Листинг 22: roles/caddy/templates/index.html.j2**

```jinja2
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Добро пожаловать на {{ caddy_domains_config[0].domain }}</title>
  <style>
    :root {
      --primary: #667eea;
      --secondary: #764ba2;
    }
    
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: linear-gradient(135deg, var(--primary) 0%, var(--secondary) 100%);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    
    .container {
      background: white;
      border-radius: 12px;
      padding: 3rem;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
      max-width: 600px;
      width: 90%;
    }
    
    h1 {
      color: #333;
      margin-bottom: 0.5rem;
      font-size: 2rem;
    }
    
    .subtitle {
      color: #666;
      margin-bottom: 2rem;
      font-size: 1.1rem;
    }
    
    .server-info {
      background: #f5f5f5;
      border-left: 4px solid var(--primary);
      padding: 1rem;
      margin: 1.5rem 0;
      border-radius: 4px;
    }
    
    .server-info p {
      margin: 0.5rem 0;
      color: #555;
    }
    
    .server-info strong {
      color: #333;
    }
    
    .domains {
      margin: 2rem 0;
    }
    
    .domains h2 {
      color: #333;
      font-size: 1.3rem;
      margin-bottom: 1rem;
    }
    
    .domain-list {
      display: flex;
      flex-wrap: wrap;
      gap: 0.5rem;
    }
    
    .domain-tag {
      background: var(--primary);
      color: white;
      padding: 0.5rem 1rem;
      border-radius: 20px;
      font-size: 0.9rem;
    }
    
    .status {
      display: flex;
      align-items: center;
      margin-top: 2rem;
      padding-top: 2rem;
      border-top: 1px solid #eee;
    }
    
    .status-indicator {
      width: 12px;
      height: 12px;
      background: #4caf50;
      border-radius: 50%;
      margin-right: 0.5rem;
      animation: pulse 2s infinite;
    }
    
    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.5; }
    }
    
    .status-text {
      color: #4caf50;
      font-weight: 600;
    }
    
    .footer {
      margin-top: 2rem;
      padding-top: 2rem;
      border-top: 1px solid #eee;
      font-size: 0.9rem;
      color: #999;
      text-align: center;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🚀 Добро пожаловать!</h1>
    <p class="subtitle">Ваш сервер Caddy успешно работает</p>
    
    <div class="server-info">
      <p><strong>Хост:</strong> {{ inventory_hostname }}</p>
      <p><strong>ОС:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
      <p><strong>Версия Caddy:</strong> {{ caddy_version }}</p>
      <p><strong>Дата развёртывания:</strong> {{ ansible_date_time.iso8601_basic_short }}</p>
      {%- if ansible_default_ipv4.address is defined %}
      <p><strong>IP адрес:</strong> {{ ansible_default_ipv4.address }}</p>
      {%- endif %}
    </div>
    
    {%- if caddy_domains_config %}
    <div class="domains">
      <h2>Настроенные домены:</h2>
      <div class="domain-list">
      {%- for domain in caddy_domains_config %}
        <span class="domain-tag">{{ domain.domain }}</span>
        {%- if domain.aliases %}
          {%- for alias in domain.aliases %}
        <span class="domain-tag">{{ alias }}</span>
          {%- endfor %}
        {%- endif %}
      {%- endfor %}
      </div>
    </div>
    {%- endif %}
    
    <div class="status">
      <div class="status-indicator"></div>
      <span class="status-text">Сервис активен и работает нормально</span>
    </div>
    
    <div class="footer">
      <p>Управляется: Ansible + Caddy | Advanced Architecture</p>
      <p>Последнее обновление: {{ ansible_date_time.iso8601 }}</p>
    </div>
  </div>
</body>
</html>
```

### 3.3 Валидация конфигурации и тестирование перед применением

**Листинг 23: playbooks/validate_and_test.yml**

```yaml
---
- name: Validate and Test Caddy Configuration
  hosts: caddy_nodes
  gather_facts: true
  
  tasks:
    - name: Validate YAML syntax
      block:
        - name: Check Caddyfile syntax
          shell: |
            /usr/bin/caddy validate --config {{ caddy_config_dir }}/Caddyfile
          register: caddy_validate
          changed_when: false
        
        - name: Display validation results
          debug:
            msg: "{{ caddy_validate.stdout }}"
      
      rescue:
        - name: Handle validation errors
          debug:
            msg: "Validation failed: {{ caddy_validate.stderr }}"
          failed_when: true
    
    - name: Test connectivity
      block:
        - name: Test HTTP connectivity
          uri:
            url: "http://localhost:80"
            method: GET
            status_code: [200, 301, 302, 404]
          register: http_test
          ignore_errors: true
        
        - name: Test HTTPS connectivity
          uri:
            url: "https://localhost:443"
            validate_certs: no
            method: GET
            status_code: [200, 301, 302, 404]
          register: https_test
          ignore_errors: true
        
        - name: Display connectivity results
          debug:
            msg: |
              HTTP Status: {{ http_test.status | default('N/A') }}
              HTTPS Status: {{ https_test.status | default('N/A') }}
    
    - name: Performance tests
      block:
        - name: Measure response time
          shell: |
            time curl -s http://localhost/ > /dev/null
          register: response_time
          changed_when: false
    
    - name: Security headers validation
      block:
        - name: Check security headers
          shell: |
            curl -sI http://localhost/ | grep -E "X-Frame-Options|X-Content-Type-Options|Strict-Transport-Security"
          register: headers_check
          changed_when: false
        
        - name: Display headers
          debug:
            msg: "{{ headers_check.stdout }}"
```

---

## Задания

### 5.1 Создание переиспользуемых playbook'ов для операций с файлов

**Листинг 24: playbooks/file_operations.yml (переиспользуемый playbook)**

```yaml
---
- name: Reusable File Operations
  hosts: all
  
  vars:
    operation: create  # create, modify, delete, backup
    target_file: /tmp/test.txt
    file_content: "Test content"
    backup_dir: /tmp/backups
  
  tasks:
    - name: Create file
      block:
        - name: Ensure directory exists
          file:
            path: "{{ target_file | dirname }}"
            state: directory
            mode: '0755'
        
        - name: Create file with content
          copy:
            content: "{{ file_content }}"
            dest: "{{ target_file }}"
            mode: '0644'
          when: operation == 'create'
      
      tags: [file_create]
    
    - name: Modify file
      block:
        - name: Update file content
          lineinfile:
            path: "{{ target_file }}"
            line: "{{ file_content }}"
            create: yes
          when: operation == 'modify'
      
      tags: [file_modify]
    
    - name: Backup file
      block:
        - name: Create backup directory
          file:
            path: "{{ backup_dir }}"
            state: directory
        
        - name: Backup file
          copy:
            src: "{{ target_file }}"
            dest: "{{ backup_dir }}/{{ target_file | basename }}.bak.{{ ansible_date_time.iso8601_basic_short }}"
            remote_src: yes
          when: operation == 'backup'
      
      tags: [file_backup]
    
    - name: Delete file
      block:
        - name: Remove file
          file:
            path: "{{ target_file }}"
            state: absent
          when: operation == 'delete'
      
      tags: [file_delete]
```

### 5.2 Расширенная конфигурация с использованием переменных и фактов

**Листинг 25: playbooks/extended_caddy_config.yml**

```yaml
---
- name: Extended Caddy Configuration with Facts
  hosts: caddy_nodes
  gather_facts: true
  
  vars:
    # Использование фактов в переменных
    server_memory_gb: "{{ (ansible_memtotal_mb / 1024) | int }}"
    server_cpu_count: "{{ ansible_processor_vcpus }}"
    deployment_timestamp: "{{ ansible_date_time.iso8601 }}"
  
  tasks:
    - name: Calculate optimal Caddy settings based on system resources
      set_fact:
        caddy_max_connections: "{{ (server_cpu_count * 1000) | int }}"
        caddy_worker_threads: "{{ server_cpu_count }}"
        caddy_cache_size_mb: "{{ (server_memory_gb * 128) | int }}"
      
    - name: Configure Caddy based on available resources
      template:
        src: caddyfile-optimized.j2
        dest: "{{ caddy_config_dir }}/Caddyfile"
        validate: "/usr/bin/caddy validate --config %s"
      vars:
        resource_optimized: true
      notify: reload caddy
    
    - name: Display configuration summary
      debug:
        msg: |
          System Resources:
          - CPU Cores: {{ server_cpu_count }}
          - Memory: {{ server_memory_gb }} GB
          - Optimized Settings:
            - Max Connections: {{ caddy_max_connections }}
            - Worker Threads: {{ caddy_worker_threads }}
            - Cache Size: {{ caddy_cache_size_mb }} MB
          - Deployment Time: {{ deployment_timestamp }}
  
  handlers:
    - name: reload caddy
      systemd:
        name: caddy
        state: reloaded
```

---

## Выводы

1. **Реализована продвинутая архитектура Ansible:**
   - Динамический inventory с использованием Python скриптов
   - Иерархия ролей с явными зависимостями (dependencies)
   - Модульное разделение ответственности

2. **Использованы продвинутые возможности Ansible:**
   - Handlers для управления жизненным циклом сервиса
   - Условное выполнение задач (when, block, rescue)
   - Циклы и фильтры Jinja2 для трансформации данных
   - Переменные разных уровней (defaults, vars, group_vars, host_vars)

3. **Реализована валидация конфигурации:**
   - Синтаксическая проверка Caddyfile
   - Функциональное тестирование (HTTP/HTTPS)
   - Проверка заголовков безопасности

4. **Созданы переиспользуемые компоненты:**
   - Playbook'ы с параметризацией
   - Универсальные шаблоны Jinja2
   - Модули для различных операций с файлами

5. **Оптимизация на основе фактов:**
   - Динамическая конфигурация на основе системных ресурсов
   - Расчёт оптимальных параметров Caddy
   - Адаптивная настройка в зависимости от окружения

6. **Гибкая управление доменами:**
   - Динамическая регистрация доменов
   - Условная генерация конфигурации
   - Поддержка расширенных параметров (aliases, заголовки безопасности)

7. **Тестирование и мониторинг:**
   - Валидация конфигурации перед применением
   - Функциональное тестирование после развёртывания
   - Проверка состояния сервиса

---

## Приложение A: Полная структура проекта

```
ansible-caddy-advanced/
├── ansible.cfg
├── inventory/
│   ├── static_hosts.yml
│   ├── dynamic_inventory.py
│   ├── group_vars/
│   │   └── caddy_nodes.yml
│   └── host_vars/
│       └── localhost.yml
├── roles/
│   ├── caddy/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── meta/
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── files/
│   ├── caddy-dependencies/
│   ├── caddy-config/
│   └── caddy-service/
├── playbooks/
│   ├── deploy_caddy.yml
│   ├── validate_environment.yml
│   ├── validate_and_test.yml
│   ├── file_operations.yml
│   └── extended_caddy_config.yml
├── molecule/
│   └── default/
│       ├── converge.yml
│       ├── verify.yml
│       └── molecule.yml
├── .ansible-lint
└── .yamllint
```

---

## Приложение B: Примеры конфигурационных файлов

### B.1 Полный ansible.cfg с комментариями

```ini
[defaults]
# Основная конфигурация
inventory = inventory/
host_key_checking = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_cache
fact_caching_timeout = 3600

# Пути и поиск модулей/ролей
library = ./library
roles_path = roles:galaxy-roles
collections_paths = ./collections

# Вывод и отладка
stdout_callback = yaml
callback_whitelist = profile_tasks, timer
force_color = True
display_skipped_hosts = False
display_ok_hosts = True

# Параметры выполнения
timeout = 10
max_fail_percentage = 0
forks = 5
async_dir = /tmp/.ansible_async

# Обработчик ошибок
action_warnings = False
deprecation_warnings = False

# Эскалация привилегий
[privilege_escalation]
become = True
become_method = sudo
become_user = root

[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s
```

---

**Конец отчёта**
