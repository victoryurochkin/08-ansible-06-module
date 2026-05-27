# Домашнее задание к занятию 6 «Создание собственных модулей» - Юрочкин В.А

## Описание

В рамках домашнего задания был создан собственный Ansible module, оформленный в виде Ansible collection.

Основная задача модуля — создавать или обновлять текстовый файл на хосте по пути, заданному параметром `path`, с содержимым, заданным параметром `content`.

Дополнительно внутри collection была создана role, которая использует этот custom module. После этого collection была собрана в архив `.tar.gz`, установлена из локального архива и проверена через отдельный playbook.

## Репозиторий

```text
https://github.com/victoryurochkin/08-ansible-06-module
```

## Git tag

```text
1.0.0
```

## Collection

```text
my_own_namespace.yandex_cloud_elk
```

## Custom module

```text
my_own_namespace.yandex_cloud_elk.my_own_module
```

## Role

```text
my_own_namespace.yandex_cloud_elk.create_file
```

## Архив collection

```text
my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

Архив находится в корне репозитория:

```text
https://github.com/victoryurochkin/08-ansible-06-module/blob/main/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

---

## Подготовка окружения

Работа выполнялась на управляющем Ansible-хосте.

Проверка версий:

```bash
ansible --version
ansible-galaxy --version
python3 --version
pip3 --version
git --version
```

Используемые версии:

```text
ansible-core 2.17.14
Python 3.10.12
pip 26.1.1
git 2.34.1
```

---

## Клонирование репозитория

Для выполнения задания был создан и использован репозиторий:

```text
https://github.com/victoryurochkin/08-ansible-06-module
```

Клонирование репозитория:

```bash
cd /root

git clone git@github.com:victoryurochkin/08-ansible-06-module.git

cd /root/08-ansible-06-module
```

---

## Инициализация Ansible collection

Collection была создана командой:

```bash
cd /root/08-ansible-06-module

ansible-galaxy collection init my_own_namespace.yandex_cloud_elk
```

После этого collection была размещена в стандартной структуре Ansible collections:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk
```

Итоговый путь collection:

```text
/root/08-ansible-06-module/ansible_collections/my_own_namespace/yandex_cloud_elk
```

---

## Структура проекта

Итоговая структура репозитория:

```text
08-ansible-06-module/
├── ansible_collections
│   └── my_own_namespace
│       └── yandex_cloud_elk
│           ├── docs
│           ├── galaxy.yml
│           ├── meta
│           │   └── runtime.yml
│           ├── my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
│           ├── plugins
│           │   ├── modules
│           │   │   └── my_own_module.py
│           │   └── README.md
│           ├── README.md
│           └── roles
│               └── create_file
│                   ├── defaults
│                   │   └── main.yml
│                   └── tasks
│                       └── main.yml
├── install_test
│   ├── my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
│   └── test_installed_collection.yml
├── my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
├── README.md
├── test_module.yml
└── test_role.yml
```

---

## Создание собственного Ansible module

Модуль был создан в каталоге:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/plugins/modules/my_own_module.py
```

Модуль принимает два обязательных параметра:

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `path` | string | да | Путь к создаваемому или обновляемому файлу |
| `content` | string | да | Содержимое файла |

Логика работы модуля:

1. Получает параметры `path` и `content`.
2. Проверяет, существует ли файл по указанному пути.
3. Если файл существует и его содержимое уже совпадает с нужным, возвращает `changed: false`.
4. Если файла нет или его содержимое отличается, создаёт или обновляет файл.
5. Возвращает `changed: true`, если файл был создан или изменён.
6. Поддерживает `check_mode`.

---

## Содержимое custom module

Файл:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/plugins/modules/my_own_module.py
```

Основная логика модуля:

```python
#!/usr/bin/python

from __future__ import absolute_import, division, print_function
__metaclass__ = type

import os
from ansible.module_utils.basic import AnsibleModule


def run_module():
    module_args = dict(
        path=dict(type='str', required=True),
        content=dict(type='str', required=True),
    )

    result = dict(
        changed=False,
        path='',
        content='',
        message='',
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True,
    )

    path = module.params['path']
    content = module.params['content']

    result['path'] = path
    result['content'] = content

    current_content = None

    if os.path.exists(path):
        try:
            with open(path, 'r', encoding='utf-8') as file:
                current_content = file.read()
        except Exception as error:
            module.fail_json(
                msg='Failed to read existing file: {0}'.format(error),
                **result
            )

    if current_content == content:
        result['changed'] = False
        result['message'] = 'File already exists with required content'
        module.exit_json(**result)

    result['changed'] = True

    if module.check_mode:
        result['message'] = 'File would be created or updated'
        module.exit_json(**result)

    directory = os.path.dirname(path)

    if directory and not os.path.exists(directory):
        try:
            os.makedirs(directory, exist_ok=True)
        except Exception as error:
            module.fail_json(
                msg='Failed to create directory: {0}'.format(error),
                **result
            )

    try:
        with open(path, 'w', encoding='utf-8') as file:
            file.write(content)
    except Exception as error:
        module.fail_json(
            msg='Failed to write file: {0}'.format(error),
            **result
        )

    result['message'] = 'File has been created or updated'
    module.exit_json(**result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

---

## Локальная проверка module

Проверка выполнялась прямым запуском Python-файла модуля.

Команда:

```bash
cd /root/08-ansible-06-module/ansible_collections/my_own_namespace/yandex_cloud_elk

python3 plugins/modules/my_own_module.py <<'EOF'
{"ANSIBLE_MODULE_ARGS": {"path": "/tmp/my_own_module_local_test.txt", "content": "Hello from local module test"}}
EOF
```

Результат первого запуска:

```json
{
  "changed": true,
  "path": "/tmp/my_own_module_local_test.txt",
  "content": "Hello from local module test",
  "message": "File has been created or updated"
}
```

Проверка содержимого файла:

```bash
cat /tmp/my_own_module_local_test.txt
```

Результат:

```text
Hello from local module test
```

Повторный запуск модуля возвращает `changed: false`, так как файл уже существует и его содержимое соответствует требуемому:

```json
{
  "changed": false,
  "path": "/tmp/my_own_module_local_test.txt",
  "content": "Hello from local module test",
  "message": "File already exists with required content"
}
```

---

## Single task playbook для проверки module

Для проверки модуля через Ansible был создан playbook:

```text
test_module.yml
```

Содержимое файла:

```yaml
---
- name: Test custom Ansible module
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create file using my own module
      my_own_namespace.yandex_cloud_elk.my_own_module:
        path: /tmp/my_own_module_playbook_test.txt
        content: "Hello from custom Ansible module"
```

Запуск playbook:

```bash
cd /root/08-ansible-06-module

ANSIBLE_COLLECTIONS_PATH=/root/08-ansible-06-module \
ansible-playbook test_module.yml
```

Проверка идемпотентности выполнялась повторным запуском:

```bash
ANSIBLE_COLLECTIONS_PATH=/root/08-ansible-06-module \
ansible-playbook test_module.yml
```

Результат повторного запуска:

```text
PLAY RECAP
localhost                  : ok=1 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Проверка содержимого файла:

```bash
cat /tmp/my_own_module_playbook_test.txt
```

Результат:

```text
Hello from custom Ansible module
```

---

## Создание role внутри collection

Single task playbook был преобразован в single task role внутри collection.

Role создана в каталоге:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/roles/create_file
```

Структура role:

```text
roles/create_file/
├── defaults
│   └── main.yml
└── tasks
    └── main.yml
```

---

## Default-переменные role

Файл:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/roles/create_file/defaults/main.yml
```

Содержимое:

```yaml
---
my_own_module_path: /tmp/my_own_module_role_test.txt
my_own_module_content: "Hello from custom Ansible role"
```

---

## Tasks role

Файл:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/roles/create_file/tasks/main.yml
```

Содержимое:

```yaml
---
- name: Create file using my own module
  my_own_namespace.yandex_cloud_elk.my_own_module:
    path: "{{ my_own_module_path }}"
    content: "{{ my_own_module_content }}"
```

---

## Playbook для проверки role

Для проверки role был создан playbook:

```text
test_role.yml
```

Содержимое:

```yaml
---
- name: Test custom role from collection
  hosts: localhost
  gather_facts: false

  roles:
    - role: my_own_namespace.yandex_cloud_elk.create_file
```

Запуск playbook:

```bash
cd /root/08-ansible-06-module

ANSIBLE_COLLECTIONS_PATH=/root/08-ansible-06-module \
ansible-playbook test_role.yml
```

Повторный запуск подтверждает идемпотентность:

```text
PLAY RECAP
localhost                  : ok=1 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Проверка файла:

```bash
cat /tmp/my_own_module_role_test.txt
```

Результат:

```text
Hello from custom Ansible role
```

---

## Документация collection

В collection заполнены основные файлы документации и метаданных:

```text
ansible_collections/my_own_namespace/yandex_cloud_elk/README.md
ansible_collections/my_own_namespace/yandex_cloud_elk/galaxy.yml
```

Файл `galaxy.yml`:

```yaml
---
namespace: my_own_namespace
name: yandex_cloud_elk
version: 1.0.0
readme: README.md
authors:
  - Viktor Yurochkin (@victoryurochkin)
description: Custom Ansible collection with module and role for creating text files.
license:
  - MIT
tags:
  - ansible
  - module
  - collection
  - file
  - homework
dependencies: {}
repository: https://github.com/victoryurochkin/08-ansible-06-module
documentation: https://github.com/victoryurochkin/08-ansible-06-module
homepage: https://github.com/victoryurochkin/08-ansible-06-module
issues: https://github.com/victoryurochkin/08-ansible-06-module/issues
build_ignore:
  - "*.tar.gz"
  - ".git"
  - ".gitignore"
```

---

## Сборка collection

Сборка collection выполнялась из корня collection:

```bash
cd /root/08-ansible-06-module/ansible_collections/my_own_namespace/yandex_cloud_elk

ansible-galaxy collection build --force
```

Результат:

```text
Created collection for my_own_namespace.yandex_cloud_elk at /root/08-ansible-06-module/ansible_collections/my_own_namespace/yandex_cloud_elk/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

Проверка архива:

```bash
ls -lh *.tar.gz
```

Результат:

```text
-rw-r--r-- 1 root root 3.8K May 27 10:17 my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

Архив также был скопирован в корень репозитория:

```text
/root/08-ansible-06-module/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
```

---

## Проверка установки collection из локального архива

Для проверки установки collection из локального архива была создана отдельная директория:

```text
install_test
```

В неё были помещены:

```text
install_test/
├── my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz
└── test_installed_collection.yml
```

---

## Playbook для проверки установленной collection

Файл:

```text
install_test/test_installed_collection.yml
```

Содержимое:

```yaml
---
- name: Test installed custom collection
  hosts: localhost
  gather_facts: false

  roles:
    - role: my_own_namespace.yandex_cloud_elk.create_file
      vars:
        my_own_module_path: /tmp/my_own_module_installed_collection_test.txt
        my_own_module_content: "Hello from installed custom collection"
```

---

## Установка collection из архива

Команда установки:

```bash
cd /root/08-ansible-06-module/install_test

ansible-galaxy collection install my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz --force
```

Результат:

```text
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing 'my_own_namespace.yandex_cloud_elk:1.0.0' to '/root/.ansible/collections/ansible_collections/my_own_namespace/yandex_cloud_elk'
my_own_namespace.yandex_cloud_elk:1.0.0 was installed successfully
```

---

## Запуск playbook с установленной collection

Первый запуск:

```bash
ansible-playbook test_installed_collection.yml
```

Результат:

```text
PLAY [Test installed custom collection]

TASK [my_own_namespace.yandex_cloud_elk.create_file : Create file using my own module]
changed: [localhost]

PLAY RECAP
localhost                  : ok=1 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Повторный запуск:

```bash
ansible-playbook test_installed_collection.yml
```

Результат:

```text
PLAY [Test installed custom collection]

TASK [my_own_namespace.yandex_cloud_elk.create_file : Create file using my own module]
ok: [localhost]

PLAY RECAP
localhost                  : ok=1 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Проверка содержимого созданного файла:

```bash
cat /tmp/my_own_module_installed_collection_test.txt
```

Результат:

```text
Hello from installed custom collection
```

---

## Фиксация результата в Git

После выполнения всех этапов изменения были добавлены в Git:

```bash
cd /root/08-ansible-06-module

git add .
git commit -m "Add custom ansible module collection"
git tag 1.0.0

git push origin main
git push origin 1.0.0
```

Проверка истории:

```bash
git log --oneline --decorate -5
```

Результат:

```text
1da6f6d (HEAD -> main, tag: 1.0.0, origin/main, origin/HEAD) Add custom ansible module collection
9932332 Initial commit
```

Проверка тегов:

```bash
git tag
```

Результат:

```text
1.0.0
```

---

## Итог

В результате выполнения задания:

- создан собственный Ansible module `my_own_module`;
- module принимает параметры `path` и `content`;
- module создаёт или обновляет текстовый файл;
- module поддерживает идемпотентность;
- выполнена локальная проверка module;
- создан single task playbook `test_module.yml`;
- проверена идемпотентность через playbook;
- создана Ansible collection `my_own_namespace.yandex_cloud_elk`;
- module перенесён в collection;
- single task playbook преобразован в role `create_file`;
- для role созданы default-переменные;
- создан playbook `test_role.yml` для использования role;
- collection задокументирована;
- collection собрана в архив `my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz`;
- создана директория `install_test` для проверки установки;
- collection установлена из локального архива;
- playbook с установленной collection успешно запущен;
- проверена идемпотентность установленной collection;
- создан git tag `1.0.0`;
- результат загружен в GitHub.

---

## Ссылки для проверки

| Назначение | Ссылка |
|---|---|
| Репозиторий | <https://github.com/victoryurochkin/08-ansible-06-module> |
| Архив collection | <https://github.com/victoryurochkin/08-ansible-06-module/blob/main/my_own_namespace-yandex_cloud_elk-1.0.0.tar.gz> |
| Git tag 1.0.0 | <https://github.com/victoryurochkin/08-ansible-06-module/releases/tag/1.0.0> |
| Custom module | <https://github.com/victoryurochkin/08-ansible-06-module/blob/main/ansible_collections/my_own_namespace/yandex_cloud_elk/plugins/modules/my_own_module.py> |
| Role create_file | <https://github.com/victoryurochkin/08-ansible-06-module/tree/main/ansible_collections/my_own_namespace/yandex_cloud_elk/roles/create_file> |
