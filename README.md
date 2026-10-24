# ДЗ №1. Изучение средств защиты ОС GNU/Linux

**Система:** РЕД ОС 8.0 (RPM/DNF), SELinux в режиме Enforcing
**Виртуализация:** QEMU/KVM, virt-manager
**Пользователь:** user1, UID 1234, группа students

| Раздел | Требование задания | Как выполнено | Артефакт |
|---|---|---|---|
| 1 | Создать user1 с идентификатором 1234 | `groupadd students`; `useradd -m -u 1234 -s /bin/bash -G students user1` | etc-passwd, etc-group, etc-shadow, stat.out |
| 1 | Вход в группу students | ключ `-G` (дополнительные группы) в `useradd` | etc-group, id user1 в history.out |
| 1 | Смена пароля каждые 3 месяца | `chage -m 1 -M 90 -W 7 -I 3 user1` | etc-shadow (поля 5-8) |
| 2.1 | Найти все файлы с битом set-UID | `find / \( -path /proc -o -path /sys -o -path /dev \) -prune -o -type f -perm -4000 -print` | find-setuid.txt |
| 2.2 | Процессы с EUID=0 при RUID!=0 | `ps -eo pid,ruid,euid,user,comm` + awk; `passwd` запущен под user1 | process-elevated.txt |
| 3 | set-UID-механизм | копия bash в /home/user1, `chown root:root`, `chmod 4755` | stat.out |
| 4 | Механизм привилегий (capabilities) | копия chown, `setcap cap_chown+ep` | getcap.out, stat.out |
| 5 | sudo на изменение системного времени | правило в /etc/sudoers.d/change-time | etc-sudoers, sudoers-change-time |
| 6 | Сбор артефактов | history.out, stat.out, getcap.out, скриншот | все файлы |
| 7 | Публикация на GitHub | git clone/add/commit/push | репозиторий |

## Наблюдения

**Различие RUID и EUID** — ключевой механизм задания. Запуск setuid-копии от имени
user1 дал RUID=1234 при EUID=0: процесс инициирован обычным пользователем, но
выполняется от root.

**Capabilities не наследуются дочерними процессами.** Capability на копии bash
не передалась бы вложенной команде chown, поэтому capability задана на копии
самого chown. Механизмы setuid и capabilities взаимоисключающие: setcap сбрасывает
бит setuid, поэтому в разделах 3 и 4 использованы разные копии.

**SELinux в режиме Enforcing.** Бит setuid работает, но метка файла должна быть
корректной. Унаследованная от `sudo cp` метка `unconfined_u` приведена к канонической
через `restorecon`. Демонстрация выполнялась в неограниченном домене (sudo), поэтому
ограничения SELinux к ней не применялись.

**/home — отдельная файловая система.** Опция `nosuid` не установлена, поэтому бит
setuid в /home/user1 работает. Поиск setuid-файлов выполнен с `-prune` по /proc, /sys
и /dev — иначе обход виртуальных ФС даёт тысячи служебных записей.

**Контрольный провал в разделе 4.** Обычный `chown` от user1 на файл, принадлежащий
root, завершается ошибкой "Операция не позволена", тогда как копия с capability
выполняет ту же операцию. Это подтверждает, что именно capability обеспечивает успех.

**Привилегированная операция в разделе 3** — чтение `/etc/shadow` (права 000,
владелец root). В примере задания указан `/etc/passwd`, но он имеет права 644 и
доступен любому пользователю, поэтому root для его чтения не требуется.

## Формат системных файлов

Файлы `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers` выгружены в
репозиторий под именами `etc-passwd`, `etc-shadow`, `etc-group`, `etc-sudoers` —
Git не допускает путей с ведущим слешем. Содержимое не изменялось.
