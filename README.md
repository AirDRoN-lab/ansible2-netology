# Домашнее задание к занятию "08.05 Тестирование Roles"

## Подготовка к выполнению
1. Установите molecule: `pip3 install "molecule==3.5.2"`
2. Выполните `docker pull aragast/netology:latest` -  это образ с podman, tox и несколькими пайтонами (3.7 и 3.9) внутри

```
vagrant@server1:~/ansible2-netology$ molecule --version
molecule 3.4.0 using python 3.8
    ansible:2.12.6
    delegated:3.4.0 from molecule
    docker:1.1.0 from molecule_docker requiring collections: community.docker>=1.9.1

vagrant@server1:~/ansible2-netology$ docker image ls | grep aragast
aragast/netology                             latest    1a7ccc2470f9   5 weeks ago     2.46GB
```

## Основная часть

Наша основная цель - настроить тестирование наших ролей. Задача: сделать сценарии тестирования для vector. Ожидаемый результат: все сценарии успешно проходят тестирование ролей.

### Molecule

1. Запустите  `molecule test -s centos7` внутри корневой директории clickhouse-role, посмотрите на вывод команды.

Тестирование выполнено успешно (но команда должны быть `molecule test -s centos_7`). Вывод не прилагаю. 

2. Перейдите в каталог с ролью vector-role и создайте сценарий тестирования по умолчанию при помощи `molecule init scenario --driver-name docker`.

Centos не создавался, т.к. таски написаны под debian/ubuntu. 
```
vagrant@server1:~/ansible2-netology/playbook/roles/vector$ molecule init scenario --driver-name docker ubuntu
INFO     Initializing new scenario ubuntu...
INFO     Initialized scenario in /home/vagrant/ansible2-netology/playbook/roles/vector/molecule/ubuntu successfully.
```

3. Добавьте несколько разных дистрибутивов (centos:8, ubuntu:latest) для инстансов и протестируйте роль, исправьте найденные ошибки, если они есть.

Роль была полностью переписана, так как не проходила тест на идемпотентность. Копирование файлов, разархивирование скачанного архива. Тем не менее при установки Vector выполняется скрипт, который выполняет проверку на зависимости и добавляет репозиторий apt, для него добавлен`changed_when: false`.
Роль выложениа в текущий репозиторий.

4. Добавьте несколько assert'ов в verify.yml файл для  проверки работоспособности vector-role (проверка, что конфиг валидный, проверка успешности запуска, etc). Запустите тестирование роли повторно и проверьте, что оно прошло успешно.

Добавлена проверка в verify.yml.

```
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
  - name: Check Vector process
    ansible.builtin.shell: 
      cmd: 'systemctl status vector.service'
    register: vector_service_status
  - name: Check Vector version
    ansible.builtin.shell: 
      cmd: 'vector --version'
    register: vector_version_status  
  - name: Check Vector data_dir
    ansible.builtin.shell: 
      cmd: 'grep ^data_dir /etc/vector/vector.toml'
  - name: Check release codes from Vector checking 
    assert:
      that: 
        - vector_service_status.rc == 0
        - vector_version_status.rc == 0
```

5. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

Добавлен тэг соответствующий уроку 08-ansible-05-testing

### Tox

1. Добавьте в директорию с vector-role файлы из [директории](./example)

Добавлено в корень роли:
```
vagrant@server1:~/ansible2-netology/playbook/roles/vector$ ll
total 60
drwxrwxr-x 11 vagrant vagrant 4096 Jul 23 15:47 ./
drwxrwxr-x  5 vagrant vagrant 4096 Jul  1 18:57 ../
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 defaults/
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 handlers/
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 meta/
drwxrwxr-x  5 vagrant vagrant 4096 Jul 23 16:45 molecule/
-rw-rw-r--  1 vagrant vagrant 1328 Jul  1 18:28 README.md
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 tasks/
drwxrwxr-x  2 vagrant vagrant 4096 Jul 23 09:48 templates/
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 tests/
drwxr-xr-x  7 root    root    4096 Jul 23 16:26 .tox/
-rw-rw-r--  1 vagrant vagrant  284 Jul 23 15:44 tox.ini
-rw-rw-r--  1 vagrant vagrant   90 Jul 23 15:44 tox-requirements.txt
drwxrwxr-x  2 vagrant vagrant 4096 Jul  1 18:57 vars/
-rw-rw-r--  1 vagrant vagrant  598 Jul 23 07:32 .yamllint
```

2. Запустите `docker run --privileged=True -v <path_to_repo>:/opt/vector-role -w /opt/vector-role -it aragast/netology:latest /bin/bash`, где path_to_repo - путь до корня репозитория с vector-role на вашей файловой системе.

```
docker run --privileged=True -v /home/vagrant/ansible2-netology/playbook/roles/vector:/opt/vector-role -w /opt/vector-role -it aragast/netology:latest /bin/bash
[root@e2f67ce49877 vector-role]#
```

3. Внутри контейнера выполните команду `tox`, посмотрите на вывод.

Перед выполнением изменена команда в tox.ini на `{posargs:molecule test -s ubuntu --destroy always}`
Исполнение команды крайне медленное (более 2 часов), принято решение облегчить роль, не собирать докер контейнер и соответсвенно дописать проверку `when: ansible_virtualization_type != "docker"`, чтобы не запускать внутри контейнера systemd.

5. Создайте облегчённый сценарий для `molecule` с драйвером `molecule_podman`. Проверьте его на исполнимость.

Создан, выложен здесь.

6. Пропишите правильную команду в `tox.ini` для того чтобы запускался облегчённый сценарий.

Сценарий называется compatibility, соответсвенно в tox.ini пишем `{posargs:molecule test -s compatibility --destroy always}`

8. Запустите команду `tox`. Убедитесь, что всё отработало успешно.

Все, ок. 

9. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

Добавлен тэг соответствующий уроку 08-ansible-05-testing

После выполнения у вас должно получится два сценария molecule и один tox.ini файл в репозитории. Ссылка на репозиторий являются ответами на домашнее задание. Не забудьте указать в ответе теги решений Tox и Molecule заданий.

Оба решения в одном репозитории. Сценарии лежат в разных папках molecule/compatibility и molecule/ubuntu

## Необязательная часть

1. Проделайте схожие манипуляции для создания роли lighthouse.
2. Создайте сценарий внутри любой из своих ролей, который умеет поднимать весь стек при помощи всех ролей.
3. Убедитесь в работоспособности своего стека. Создайте отдельный verify.yml, который будет проверять работоспособность интеграции всех инструментов между ними.
4. Выложите свои roles в репозитории. В ответ приведите ссылки.

Будет выполнено позже, после устранения хвостов по ДЗ
