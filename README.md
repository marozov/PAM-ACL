Домашнее задание
PAM

Описание инструкция выполнения домашнего задания:

Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников

дать конкретному пользователю права работать с докером и возможность рестартить докер сервис


Этот способ действует только для пользователя, здесь нельзя указать группу, в которую входит пользователь.
Чтобы у пользователя не было возможности войти в систему не только через SSH, а также через обычную консоль (монитор) то необходимо откорректировать два файла.

Заходим в файлы nano /etc/pam.d/sshd и nano /etc/pam.d/login и приводим их к следующему виду:

```
cat /etc/pam.d/sshd

#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account required pam_access.so # Это модуль, который может давать доступ одной группе и не давать другой, но он не может это сделать по дням и времени
account required pam_time.so # Добавляем вот эту строку для ограничения по пользователю
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
```
cat /etc/pam.d/login
#%PAM-1.0
auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
auth       substack     system-auth
auth       include      postlogin
account required pam_access.so # Это модуль, который может давать доступ одной группе и не давать другой, но он не может это сделать по дням и времени
account   required   pam_time.so # Добавляем вот эту строку для ограничения по пользователю
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_console.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```

Далее заходим в файл nano /etc/security/time.conf и добавляем в конце файла строку *;*;docker;!Tu

Создаем пользователя командой useradd docker и задаем ему пароль passwd docker
Пытаемся зайти в тот день, когда у нас работает правило ssh docker@localhost и получаем:
```
ssh docker@localhost
docker@localhost's password:
Authentication failed.
```
Второй способ это ограничения по группе.
Создаем группу groupadd admin
Добавляем туда пользователя usermod -aG admin docker
Устанавливаем компонент, с помощью которого мы сможем описывать правила входа в виде обычного bash скрипта yum install pam_script -y
Можем посмотреть какие файлы использует этот компонент rpm -ql pam_script

```
[vagrant@otuslinux ~]$ rpm -ql pam_script
/etc/pam-script.d
/etc/pam_script
/etc/pam_script_acct
/etc/pam_script_auth
/etc/pam_script_passwd
/etc/pam_script_ses_close
/etc/pam_script_ses_open
/lib64/security/pam_script.so
/usr/share/doc/pam_script-1.1.8
/usr/share/doc/pam_script-1.1.8/AUTHORS
/usr/share/doc/pam_script-1.1.8/COPYING
/usr/share/doc/pam_script-1.1.8/ChangeLog
/usr/share/doc/pam_script-1.1.8/NEWS
/usr/share/doc/pam_script-1.1.8/README
/usr/share/doc/pam_script-1.1.8/README.pam_script
/usr/share/man/man7/pam-script.7.gz
```
Приводим файл /etc/pam.d/sshd к следующему виду и добавляем строку auth required pam_script.so:
```
#%PAM-1.0

auth       required     pam_script.so # Добавляем сюда эту строку
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account required pam_access.so
account required pam_time.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```
Далее приводим файл /etc/pam_script к следующему виду: (у файла должны быть права на исполнение)
```
#!/bin/bash

if [[ `grep $PAM_USER /etc/group | grep 'admin'` ]]
then
exit 0
fi
if [[ `date +%u` > 5 ]]
then
exit 1
fi
```
В этом файле мы проверяем состоит ли пользователь в группе admin и если да то пускаем его (значит его можно пускать всегда). Если он не состоит в этой группе то срабатывает проверка на то, какой сейчас день недели, если он больше 5, т.е. выходные то не пускаем. Задача выполнена.


Дать конкретному пользователю docker права рута: 

Полностью и просто стандартными средствами это сделать нельзя, т.е. можно взломать файл passwd но это не красивый способ. 
Есть два варианта решения вопроса, первый:

grep docker /etc/passwd - Ищем нашего пользователя, которому надо дать права root

```
grep docker /etc/passwd
docker:x:1001:1001::/home/docker:/bin/bash
```
Далее меняем два значения 1001 на 0(значения uid и gid root) Теперь когда пользователь будет заходить по своему логину, у него будут права рута
Также для того чтобы работать с файлами рута, надо выполнить команду usermod -a -G root docker - Добавляем пользователя в группу root

Есть второй стандартный способ, это добавить пользователя в sudoers, т.е. дать пользователю право выполнять команду от имени root. Здесь все просто, необходимо добавить пользователя в группу wheel командой usermod -aG wheel docker, после чего можно выполнять команды с приставкой sudo
