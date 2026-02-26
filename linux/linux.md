# Linux — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### Процессы и управление
1. [Как найти и завершить процесс, занимающий определённый порт?](#1-как-найти-и-завершить-процесс-занимающий-определённый-порт)
2. [В чём разница между kill, kill -9 и kill -15?](#2-в-чём-разница-между-kill-kill--9-и-kill--15)
3. [Что такое zombie-процесс и как с ним бороться?](#3-что-такое-zombie-процесс-и-как-с-ним-бороться)
4. [Как работает strace и когда его использовать?](#4-как-работает-strace-и-когда-его-использовать)
5. [Как ограничить ресурсы процесса с помощью ulimit и cgroups?](#5-как-ограничить-ресурсы-процесса-с-помощью-ulimit-и-cgroups)

### Файловая система
6. [Что такое inode? Как диагностировать проблему с исчерпанием inode?](#6-что-такое-inode-как-диагностировать-проблему-с-исчерпанием-inode)
7. [Как найти большие файлы и директории?](#7-как-найти-большие-файлы-и-директории)
8. [В чём разница между hard link и soft link?](#8-в-чём-разница-между-hard-link-и-soft-link)
9. [Как работает lsof и в каких сценариях он полезен?](#9-как-работает-lsof-и-в-каких-сценариях-он-полезен)
10. [Что такое SUID, SGID и Sticky bit?](#10-что-такое-suid-sgid-и-sticky-bit)

### Производительность и мониторинг
11. [Как диагностировать высокую нагрузку на CPU?](#11-как-диагностировать-высокую-нагрузку-на-cpu)
12. [Как диагностировать проблемы с памятью?](#12-как-диагностировать-проблемы-с-памятью)
13. [Что такое load average и как его интерпретировать?](#13-что-такое-load-average-и-как-его-интерпретировать)
14. [Как диагностировать проблемы с дисковым I/O?](#14-как-диагностировать-проблемы-с-дисковым-io)
15. [Что такое OOM Killer и как он работает?](#15-что-такое-oom-killer-и-как-он-работает)

### Сеть
16. [Как диагностировать сетевые проблемы с помощью ss, netstat, tcpdump?](#16-как-диагностировать-сетевые-проблемы-с-помощью-ss-netstat-tcpdump)
17. [Как работает iptables? Как просматривать и добавлять правила?](#17-как-работает-iptables-как-просматривать-и-добавлять-правила)

### systemd и логи
18. [Как управлять сервисами через systemd?](#18-как-управлять-сервисами-через-systemd)
19. [Как работать с journalctl? Основные сценарии использования.](#19-как-работать-с-journalctl-основные-сценарии-использования)
20. [Как определить причину перезагрузки сервера?](#20-как-определить-причину-перезагрузки-сервера)

### Дополнительно
21. [Как работает /proc filesystem?](#21-как-работает-proc-filesystem)
22. [Что такое swap и когда его использование — проблема?](#22-что-такое-swap-и-когда-его-использование--проблема)
23. [Как работает sudo и как настраивать /etc/sudoers?](#23-как-работает-sudo-и-как-настраивать-etcsudoers)

---

## Процессы и управление

### 1. Как найти и завершить процесс, занимающий определённый порт?

Это одна из самых частых задач при дебаге. Есть несколько способов.

**Способ 1: `ss` (современный инструмент, предпочтительный)**
```bash
ss -tulnp | grep :8080
```
- `-t` — TCP
- `-u` — UDP
- `-l` — только слушающие (listening) сокеты
- `-n` — не резолвить имена (быстрее)
- `-p` — показать процесс (PID и имя)

Вывод будет содержать PID, например `users:(("nginx",pid=1234,fd=6))`.

**Способ 2: `lsof`**
```bash
lsof -i :8080
```
Покажет полную информацию: процесс, PID, пользователь, тип соединения.

**Способ 3: `fuser`**
```bash
fuser 8080/tcp
```
Возвращает только PID. Можно сразу убить:
```bash
fuser -k 8080/tcp
```

**Завершение процесса по PID:**
```bash
kill -15 <PID>   # мягкое завершение (SIGTERM)
kill -9 <PID>    # принудительное завершение (SIGKILL), если -15 не помогло
```

**Одной командой найти и убить:**
```bash
kill $(lsof -t -i :8080)
```

---

### 2. В чём разница между kill, kill -9 и kill -15?

`kill` — это команда для отправки **сигналов** процессу. По умолчанию (без указания сигнала) отправляет `SIGTERM`.

| Сигнал | Номер | Команда | Поведение |
|---|---|---|---|
| `SIGTERM` | 15 | `kill <PID>` или `kill -15 <PID>` | Вежливая просьба завершить работу. Процесс **может** перехватить сигнал, завершить текущие операции, закрыть файлы, сохранить состояние — и только потом завершиться. |
| `SIGKILL` | 9 | `kill -9 <PID>` | Безусловное уничтожение процесса ядром ОС. Процесс **не может** перехватить или игнорировать этот сигнал. Состояние не сохраняется. |
| `SIGHUP` | 1 | `kill -1 <PID>` | Изначально — сигнал разрыва соединения. Многие демоны (nginx, sshd) перечитывают конфиг при получении этого сигнала, не перезапускаясь. |

**Правило:** всегда начинай с `SIGTERM` (`kill -15`). Дай процессу 5-10 секунд на graceful shutdown. Если не помогло — используй `SIGKILL`.

```bash
# Правильный паттерн
kill -15 <PID>
sleep 5
kill -9 <PID> 2>/dev/null  # на случай если процесс уже завершился
```

> **Важно:** `SIGKILL` нельзя отправить zombie-процессу и ядерным потокам — они не имеют пользовательского контекста.

---

### 3. Что такое zombie-процесс и как с ним бороться?

**Zombie-процесс** (defunct) — это процесс, который завершил выполнение, но его запись в таблице процессов ещё существует. Это происходит потому, что родительский процесс ещё не вызвал `wait()` для считывания кода возврата дочернего.

**Как обнаружить:**
```bash
ps aux | grep 'Z'
# или
ps aux | awk '$8 == "Z"'
```

В выводе `ps` у таких процессов в колонке `STAT` стоит `Z`, а в `COMMAND` — `[имя] <defunct>`.

**Почему zombie — это проблема?**
Сами по себе они почти не потребляют ресурсы (только запись в таблице процессов). Но если их накопится очень много — таблица процессов переполнится (обычно лимит ~32768), и система не сможет создавать новые процессы.

**Как бороться:**

1. **Найти родительский процесс (PPID):**
   ```bash
   ps -o pid,ppid,stat,comm | grep Z
   ```

2. **Перезапустить родительский процесс** — это самый правильный способ. Родитель корректно вызовет `wait()` или завершится сам, после чего zombie будет усыновлён `init`/`systemd` и убран.

3. **Отправить SIGCHLD родителю** (сигнал "забери дочерний процесс"):
   ```bash
   kill -SIGCHLD <PPID>
   ```

4. **Если родитель сам висит** — убей его (`kill -9 <PPID>`). Тогда zombie перейдут под `init` (PID 1), который корректно их подберёт.

> **На собеседовании важно:** `kill -9` к самому zombie-процессу не поможет — он уже мёртв, ядро просто ещё не убрало запись.

---

### 4. Как работает strace и когда его использовать?

`strace` — утилита для **трассировки системных вызовов** (syscalls), которые выполняет процесс. По сути, она перехватывает все обращения процесса к ядру Linux: открытие файлов, сетевые операции, работу с памятью, сигналы.

**Базовое использование:**
```bash
# Запустить программу под strace
strace ls /tmp

# Присоединиться к уже запущенному процессу
strace -p <PID>

# Сохранить вывод в файл (обязательно при анализе живых сервисов)
strace -p <PID> -o /tmp/strace.log

# Показывать время выполнения каждого syscall
strace -T -p <PID>

# Статистика по syscall'ам (какой сколько раз вызван и сколько времени занял)
strace -c ls /tmp
```

**Фильтрация по конкретным syscall'ам:**
```bash
# Следить только за вызовами read/write
strace -e trace=read,write -p <PID>

# Только файловые операции
strace -e trace=file -p <PID>

# Только сетевые операции
strace -e trace=network -p <PID>
```

**Практические сценарии использования:**

| Проблема | Что делаешь |
|---|---|
| Приложение зависает, непонятно где | `strace -p <PID>` — смотришь на каком syscall оно застряло |
| Приложение не стартует, нет нормальных логов | `strace ./app 2>&1 \| head -50` — видишь какой файл не может открыть |
| Подозрение на проблемы с правами доступа | `strace -e openat ./app` — видишь EACCES или ENOENT |
| Приложение работает медленно | `strace -T -c ./app` — видишь какие syscall'ы отнимают больше всего времени |

**Пример вывода — процесс не может открыть файл конфига:**
```
openat(AT_FDCWD, "/etc/myapp/config.yaml", O_RDONLY) = -1 ENOENT (No such file or directory)
```
Сразу ясно: файл не существует по этому пути.

> **Важно:** `strace` добавляет значительный overhead на production. Используй с осторожностью и только когда необходимо. Для продакшна рассмотри `perf` или eBPF-инструменты.

---

### 5. Как ограничить ресурсы процесса с помощью ulimit и cgroups?

**`ulimit`** — ограничения на уровне shell-сессии и пользователя.

```bash
# Посмотреть текущие лимиты
ulimit -a

# Ключевые параметры:
ulimit -n 65536      # максимальное количество открытых файловых дескрипторов
ulimit -u 4096       # максимальное количество процессов пользователя
ulimit -s unlimited  # размер стека
ulimit -c unlimited  # размер core dump файла (0 = отключён)
```

Постоянные ограничения настраиваются в `/etc/security/limits.conf`:
```
# формат: <user> <soft|hard> <type> <value>
www-data  soft  nofile  65536
www-data  hard  nofile  65536
*         soft  nproc   4096
```
- **soft** — текущий лимит (процесс может повысить его сам, но не выше hard)
- **hard** — абсолютный потолок (только root может увеличить)

---

**`cgroups` (Control Groups)** — более мощный механизм ядра для ограничения ресурсов целых групп процессов.

Современные системы используют **cgroups v2**. В большинстве случаев ты взаимодействуешь с ними через `systemd`.

**Ограничение ресурсов через systemd (рекомендуемый способ):**

В unit-файле сервиса (`/etc/systemd/system/myapp.service`):
```ini
[Service]
ExecStart=/usr/bin/myapp
# Ограничение CPU (50% от одного ядра)
CPUQuota=50%
# Ограничение памяти (512 МБ, при превышении — OOM kill)
MemoryMax=512M
# Ограничение I/O
IOWeight=50
# Ограничение файловых дескрипторов
LimitNOFILE=65536
```

**Ограничение через Docker** (который тоже использует cgroups под капотом):
```bash
docker run --cpus="1.5" --memory="512m" --memory-swap="1g" nginx
```

**Просмотр текущих cgroup-лимитов для процесса:**
```bash
# Найти cgroup процесса
cat /proc/<PID>/cgroup

# Посмотреть лимиты памяти
cat /sys/fs/cgroup/memory/<cgroup-path>/memory.limit_in_bytes
```

---

## Файловая система

### 6. Что такое inode? Как диагностировать проблему с исчерпанием inode?

**Inode (index node)** — структура данных в файловой системе, которая хранит **метаданные файла**, но не его имя и не содержимое:
- тип файла
- права доступа (chmod)
- владелец (UID/GID)
- размер
- временные метки (atime, mtime, ctime)
- количество hard-ссылок
- указатели на блоки данных на диске

**Имя файла** хранится в директории и указывает на inode-номер. Поэтому hard link — это просто ещё одна запись в директории, указывающая на тот же inode.

**Проблема исчерпания inode:**
Диск может быть физически не заполнен, но при попытке создать файл получишь ошибку `No space left on device`. Причина — закончились inodes (их количество фиксируется при создании ФС).

**Диагностика:**
```bash
# Посмотреть использование inode по всем смонтированным ФС
df -i

# Вывод:
# Filesystem      Inodes  IUsed   IFree IUse% Mounted on
# /dev/sda1      6553600 6553600      0  100% /

# Найти директорию с наибольшим количеством файлов
find / -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n | tail -10
```

**Типичные причины:**
- Огромное количество мелких файлов (кэш, сессии PHP, mail-очереди)
- Незаочищенные временные файлы в `/tmp`
- Логи, разбитые на тысячи мелких файлов

**Решение:**
```bash
# Найти директорию-виновника и очистить её
# Например, PHP-сессии:
ls /var/lib/php/sessions/ | wc -l  # если миллионы — вот причина
find /var/lib/php/sessions/ -type f -delete
```

> **На sobеседовании:** хорошо добавить — при создании ФС количество inode задаётся параметром `-i bytes-per-inode` в `mkfs.ext4`. На XFS inodes растут динамически, поэтому эта проблема там редкость.

---

### 7. Как найти большие файлы и директории?

```bash
# Топ-10 самых больших файлов в текущей директории (рекурсивно)
find . -type f -printf '%s %p\n' | sort -rn | head -10

# Более наглядный вариант с человекочитаемым размером
find . -type f -exec du -h {} + | sort -rh | head -10

# Размер директорий (без рекурсии в поддиректории)
du -h --max-depth=1 /var | sort -rh

# Общий размер директории
du -sh /var/log

# Найти файлы больше 1 ГБ
find / -type f -size +1G -exec ls -lh {} \;

# Найти файлы, изменённые за последние 24 часа и больше 100 МБ
find /var/log -mtime -1 -size +100M
```

**Практический сценарий — диск заполнен, ищем виновника:**
```bash
# 1. Смотрим какой раздел заполнен
df -h

# 2. Идём в эту точку монтирования и ищем большие директории
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -10

# 3. Спускаемся в самую большую и повторяем
du -h --max-depth=1 /var | sort -rh | head -10

# 4. Находим конкретные файлы
du -h --max-depth=1 /var/log | sort -rh | head -10
```

**Важный момент — удалённые файлы, которые ещё занимают место:**
```bash
# Файл удалён, но процесс держит дескриптор открытым — место не освобождается
lsof | grep deleted
# Решение: перезапустить процесс или отправить ему сигнал для переоткрытия логов
```

---

### 8. В чём разница между hard link и soft link?

| Характеристика | Hard Link | Soft Link (Symlink) |
|---|---|---|
| Указывает на | inode (данные) | путь к файлу |
| Другая ФС | нет | да |
| На директории | нет (обычно) | да |
| При удалении оригинала | файл остаётся доступен | ссылка становится битой |
| `ls -l` | `-` (обычный файл) | `l` (lrwxrwxrwx) |

```bash
# Создание hard link
ln /etc/hosts /tmp/hosts_hard

# Создание symlink
ln -s /etc/hosts /tmp/hosts_soft

# Проверка: у hard link тот же inode, у symlink - другой
ls -li /etc/hosts /tmp/hosts_hard /tmp/hosts_soft
# 123456 -rw-r--r-- 2 root root 174 Jan 1 /etc/hosts       <- inode 123456, nlinks=2
# 123456 -rw-r--r-- 2 root root 174 Jan 1 /tmp/hosts_hard  <- тот же inode
# 654321 lrwxrwxrwx 1 root root  10 Jan 1 /tmp/hosts_soft -> /etc/hosts
```

**Практическое применение symlink в DevOps:**
```bash
# Переключение версий приложения без downtime
ln -sfn /opt/myapp/v2.1.0 /opt/myapp/current
# Nginx/приложение смотрит на /opt/myapp/current — мгновенное переключение

# Конфигурация (как в Nginx sites-enabled)
ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/
```

---

### 9. Как работает lsof и в каких сценариях он полезен?

`lsof` (List Open Files) — показывает все открытые файловые дескрипторы в системе: обычные файлы, директории, сокеты, пайпы, устройства.

```bash
# Все открытые файлы (вывод огромный, обычно с grep)
lsof

# Файлы, открытые конкретным процессом
lsof -p <PID>

# Файлы, открытые конкретным пользователем
lsof -u nginx

# Кто использует конкретный файл
lsof /var/log/nginx/access.log

# Все сетевые соединения
lsof -i

# Соединения по конкретному порту
lsof -i :80
lsof -i :80 -i :443

# TCP-соединения в статусе LISTEN
lsof -i TCP -s TCP:LISTEN

# Удалённые файлы, которые ещё открыты (занимают место!)
lsof | grep deleted
```

**Практические сценарии:**

1. **Диск заполнен, но большие файлы не видны** — удалённый файл держится открытым процессом:
   ```bash
   lsof | grep deleted | awk '{print $7, $1, $2}' | sort -rn | head -10
   # Размер, процесс, PID — перезапускаем виновника
   ```

2. **Порт занят, но ss/netstat не даёт ясности:**
   ```bash
   lsof -i :8080 -n -P
   ```

3. **Проверить, какие файлы держит зависший процесс:**
   ```bash
   lsof -p <PID> | grep -E "REG|DIR"
   ```

---

### 10. Что такое SUID, SGID и Sticky bit?

Это специальные биты прав доступа в Unix/Linux, расширяющие стандартную модель `rwx`.

**SUID (Set User ID) — бит на файлах:**
Когда установлен на исполняемом файле — программа запускается с правами **владельца файла**, а не запускающего пользователя.

```bash
# Классический пример: /usr/bin/passwd
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#    ^ 's' вместо 'x' — SUID установлен

# Обычный пользователь может менять свой пароль (запись в /etc/shadow),
# потому что passwd запускается от root
```

**SGID (Set Group ID):**
- На файле: программа запускается с правами **группы владельца** файла.
- На директории: все новые файлы в ней наследуют **группу директории**, а не группу создателя. Очень полезно для shared-директорий команды.

```bash
# Установить SGID на директорию
chmod g+s /shared/project
ls -ld /shared/project
# drwxrwsr-x ... /shared/project
#       ^ 's' — SGID установлен
```

**Sticky Bit:**
На директории: удалять или переименовывать файлы может только **владелец файла** (или root), даже если у других есть право записи в директорию.

```bash
# Классический пример: /tmp
ls -ld /tmp
# drwxrwxrwt ... /tmp
#          ^ 't' — sticky bit установлен

# Установить sticky bit
chmod +t /shared/uploads
```

**Числовые значения (четвёртая цифра):**
```bash
chmod 4755 file  # SUID + rwxr-xr-x
chmod 2755 dir   # SGID + rwxr-xr-x
chmod 1777 dir   # Sticky bit + rwxrwxrwx (как /tmp)
```

**Поиск SUID/SGID файлов (важно для security audit):**
```bash
find / -perm -4000 -type f 2>/dev/null  # SUID
find / -perm -2000 -type f 2>/dev/null  # SGID
```

---

## Производительность и мониторинг

### 11. Как диагностировать высокую нагрузку на CPU?

**Шаг 1 — общая картина:**
```bash
top
# или более удобный htop
htop
```
В `top` смотрим строку `%Cpu(s)`:
- `us` — user space (приложения)
- `sy` — system/kernel
- `wa` — I/O wait (ждём диск/сеть — CPU не используется, но процессы заблокированы)
- `si` — software interrupts (сеть, высокая нагрузка может указывать на DDoS)

**Шаг 2 — найти конкретные процессы:**
```bash
# Сортировка по CPU в top: нажать 'P'
# Или через ps:
ps aux --sort=-%cpu | head -15
```

**Шаг 3 — понять что именно делает процесс:**
```bash
# Какие syscall'ы выполняет
strace -p <PID> -c  # статистика за несколько секунд, потом Ctrl+C

# Где проводит время (sampling profiler)
perf top -p <PID>
```

**Шаг 4 — если высокое `sy` (kernel CPU):**
```bash
# Посмотреть на прерывания
watch -n 1 cat /proc/interrupts

# Статистика по CPU (контекстные переключения, прерывания)
vmstat 1
# cs — context switches (если очень высокое — много потоков борются за CPU)
# in — interrupts per second
```

**Шаг 5 — если `wa` высокое (I/O wait):**
```bash
iostat -x 1
# await — среднее время ожидания I/O операции (>20ms — проблема)
# %util — загрузка устройства (>80% — насыщение)
```

**Типичные причины высокого CPU:**
- Бесконечный цикл в приложении
- Неоптимальный SQL-запрос без индекса
- Кодирование/декодирование (crypto, video)
- Утечка памяти → OOM → интенсивный swap → CPU

---

### 12. Как диагностировать проблемы с памятью?

**Общая картина:**
```bash
free -h
#               total  used   free  shared  buff/cache  available
# Mem:           15Gi  12Gi  512Mi   1.2Gi       2.8Gi      2.1Gi
```
> **Важно:** `available` — это реальная доступная память (free + освобождаемый кэш). Смотри именно на неё, а не на `free`.

**Детальная информация:**
```bash
cat /proc/meminfo
# MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapUsed...
```

**Найти процессы, потребляющие больше всего памяти:**
```bash
ps aux --sort=-%mem | head -15

# Или через smem (более точно — учитывает shared memory)
smem -r -s pss | head -15
```

**Реальное потребление памяти процессом:**
```bash
# RSS — Resident Set Size (физическая память, включая shared libs)
# PSS — Proportional Set Size (более честный показатель)
# VSZ — Virtual Size (адресное пространство, обычно огромное)
cat /proc/<PID>/status | grep -E "VmRSS|VmPSS|VmSwap"
```

**Диагностика утечки памяти:**
```bash
# Следить за ростом памяти процесса во времени
watch -n 5 'ps -p <PID> -o pid,rss,vsz,comm'

# Или через valgrind (только для dev-среды, огромный overhead)
valgrind --leak-check=full ./myapp
```

**Использование swap — тревожный сигнал:**
```bash
# Кто использует swap
for pid in /proc/[0-9]*/status; do
  awk '/^Pid|^VmSwap/{printf "%s ", $2}' "$pid"
  echo
done | sort -k2 -rn | head -10
```

---

### 13. Что такое load average и как его интерпретировать?

**Load average** — среднее количество процессов, находящихся в состоянии **готовности к выполнению** (runnable) или **ожидания** (uninterruptible sleep, т.е. ожидание I/O), за последние 1, 5 и 15 минут.

```bash
uptime
# 14:23:45 up 10 days, load average: 2.50, 1.80, 1.20
#                                     1min  5min  15min

# Также видно в top (первая строка) и в /proc/loadavg
cat /proc/loadavg
```

**Как интерпретировать:**

Ключевой вопрос — сколько у сервера **CPU ядер**:
```bash
nproc          # количество доступных ядер
nproc --all    # все ядра (включая offline)
```

- Load average = количеству ядер → **100% загрузка, очереди нет** (нормально)
- Load average > количества ядер → **есть очередь** (процессы ждут)
- Например, на 4-ядерном сервере: LA = 4.0 — норма, LA = 8.0 — очередь в 2x

**Практическая интерпретация:**

| Ситуация | Интерпретация |
|---|---|
| LA растёт: 1min > 5min > 15min | Нагрузка увеличивается прямо сейчас |
| LA падает: 1min < 5min < 15min | Нагрузка снижается, был пик |
| LA стабильно высокое | Хроническая перегрузка |

**Важный нюанс:** высокий load average не всегда означает высокий CPU. Если диск медленный, процессы могут висеть в I/O wait, поднимая LA без роста CPU.

```bash
# Различаем CPU load от I/O load:
vmstat 1
# r — runnable processes (CPU load)
# b — blocked in I/O (I/O load)
```

---

### 14. Как диагностировать проблемы с дисковым I/O?

**Основной инструмент — `iostat`:**
```bash
# Установка (пакет sysstat)
apt install sysstat

# Расширенная статистика, обновление каждую секунду
iostat -x 1

# Ключевые метрики вывода:
# r/s, w/s     — операций чтения/записи в секунду (IOPS)
# rkB/s, wkB/s — пропускная способность (КБ/с)
# await        — среднее время обработки I/O запроса (мс). >20ms — тревога
# %util        — загрузка устройства. >80% — близко к насыщению
# aqu-sz       — средняя длина очереди к устройству
```

**`iotop` — как top, но для дискового I/O:**
```bash
iotop -o  # -o показывает только активные процессы
```

**`dstat` — комплексная статистика в реальном времени:**
```bash
dstat -d -D sda 1  # только дисковые метрики для sda
dstat --top-io     # процесс с наибольшим I/O
```

**Проверить задержки на конкретном устройстве:**
```bash
# Нет нормальных данных без iostat? Используй dd для бенчмарка:
# Тест записи (очищает кэш)
dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=direct
# Тест чтения
dd if=/tmp/test of=/dev/null bs=1M iflag=direct
```

**Если проблема в конкретном процессе:**
```bash
# strace — какие файлы читает/пишет
strace -e trace=read,write,pread64,pwrite64 -p <PID>

# lsof — какие файлы открыты
lsof -p <PID> | grep REG
```

**Проверить состояние диска (SMART):**
```bash
smartctl -a /dev/sda
# Смотрим на Reallocated_Sector_Ct, Current_Pending_Sector — если не ноль, диск умирает
```

---

### 15. Что такое OOM Killer и как он работает?

**OOM Killer (Out-Of-Memory Killer)** — механизм ядра Linux, который **принудительно убивает процессы**, когда система исчерпала всю доступную память (RAM + swap) и не может выделить новую.

**Как работает:**
1. Процесс запрашивает память
2. Ядро не может её выделить
3. Ядро запускает OOM Killer
4. OOM Killer выбирает "жертву" на основе **oom_score** (0-1000)
5. Убивает процесс с наивысшим score командой SIGKILL

**Формула oom_score:** упрощённо — чем больше памяти потребляет процесс и чем менее он "важен" (не root, не ядро), тем выше score.

```bash
# Посмотреть oom_score процесса
cat /proc/<PID>/oom_score

# Посмотреть корректировку (oom_score_adj)
cat /proc/<PID>/oom_score_adj
# Значения: от -1000 (никогда не убивать) до +1000 (убивать первым)
```

**Управление приоритетом:**
```bash
# Защитить критический процесс (например, sshd) от OOM killer
echo -1000 > /proc/$(pgrep sshd)/oom_score_adj

# Через systemd (рекомендуемый способ):
# В unit-файле добавить:
# [Service]
# OOMScoreAdjust=-1000
```

**Как обнаружить, что OOM killer сработал:**
```bash
# В системных логах
dmesg | grep -i "oom\|out of memory\|killed process"
journalctl -k | grep -i oom

# Пример вывода:
# Out of memory: Kill process 12345 (java) score 892 or sacrifice child
# Killed process 12345 (java) total-vm:8192000kB, anon-rss:7654321kB
```

**Превентивные меры:**
- Настраивать `MemoryMax` в systemd unit'ах
- Использовать `cgroups` для изоляции памяти
- Мониторить `MemAvailable` в `/proc/meminfo` и алертить заранее

---

## Сеть

### 16. Как диагностировать сетевые проблемы с помощью ss, netstat, tcpdump?

**`ss` — современная замена netstat (быстрее, более детальная):**
```bash
# Все TCP соединения с PID
ss -tulnp

# Все установленные соединения
ss -tn state established

# Соединения к конкретному порту
ss -tn dst :443

# Состояния соединений (полезно при SYN flood)
ss -s
# Выведет: Total, TCP (estab, closed, orphaned, timewait)

# Найти все TIME_WAIT соединения (если их тысячи — проблема)
ss -tn state time-wait | wc -l
```

**`netstat` (старый, но ещё встречается):**
```bash
netstat -tulnp  # аналог ss -tulnp
netstat -s      # статистика по протоколам (ошибки, retransmits)
```

**`tcpdump` — перехват и анализ трафика:**
```bash
# Весь трафик на интерфейсе
tcpdump -i eth0

# Трафик от/к конкретному хосту
tcpdump -i eth0 host 192.168.1.100

# Только HTTP трафик (порт 80 и 443)
tcpdump -i eth0 port 80 or port 443

# Сохранить в файл для анализа в Wireshark
tcpdump -i eth0 -w /tmp/capture.pcap

# Показать содержимое пакетов (ASCII)
tcpdump -i eth0 -A port 80

# Диагностика: смотрим на TCP handshake и ретрансмиты
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
```

**Практический сценарий диагностики — приложение не может подключиться к БД:**
```bash
# 1. Порт открыт?
ss -tlnp | grep 5432

# 2. Можем ли достучаться с нашей машины?
telnet db-server 5432
# или
nc -zv db-server 5432

# 3. Смотрим что происходит с пакетами
tcpdump -i eth0 host db-server and port 5432

# 4. Проверяем firewall
iptables -L -n | grep 5432

# 5. Трассировка маршрута
traceroute db-server
mtr db-server  # интерактивная версия с статистикой потерь
```

---

### 17. Как работает iptables? Как просматривать и добавлять правила?

`iptables` — инструмент для управления встроенным фаерволом ядра Linux (netfilter). Пакеты проходят через цепочки (chains) таблиц (tables), где к ним применяются правила.

**Основные таблицы:**
- `filter` — основная, управляет разрешением/блокировкой трафика (цепочки: INPUT, OUTPUT, FORWARD)
- `nat` — трансляция адресов (PREROUTING, POSTROUTING, OUTPUT)
- `mangle` — модификация заголовков пакетов

**Просмотр правил:**
```bash
# Таблица filter (по умолчанию)
iptables -L -n -v
# -n — не резолвить IP/порты в имена (быстрее)
# -v — verbose (показывает счётчики пакетов и байт)
# --line-numbers — нумерация строк (нужна для удаления)
iptables -L INPUT -n -v --line-numbers

# Таблица NAT
iptables -t nat -L -n -v
```

**Добавление правил:**
```bash
# Разрешить входящий SSH (порт 22)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Разрешить входящий HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Разрешить установленные/связанные соединения (обязательное правило!)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Разрешить loopback
iptables -A INPUT -i lo -j ACCEPT

# Заблокировать конкретный IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# Блокировка с уведомлением отправителя (REJECT vs DROP)
iptables -A INPUT -s 1.2.3.4 -j REJECT --reject-with tcp-reset

# Ограничить rate (защита от брутфорса SSH)
iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/min --limit-burst 5 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

**NAT — проброс портов:**
```bash
# Включить IP forwarding (обязательно!)
echo 1 > /proc/sys/net/ipv4/ip_forward

# DNAT: пакеты на порт 8080 перенаправляем на внутренний сервер
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# MASQUERADE: для исходящего трафика (NAT для интернет-шлюза)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**Удаление правил:**
```bash
# По номеру строки
iptables -D INPUT 3

# По содержимому (заменить -A на -D)
iptables -D INPUT -p tcp --dport 22 -j ACCEPT
```

**Сохранение правил (без этого правила сбросятся после перезагрузки):**
```bash
# Debian/Ubuntu
iptables-save > /etc/iptables/rules.v4
# или
netfilter-persistent save

# RHEL/CentOS
service iptables save
```

> **На собеседовании:** стоит упомянуть `nftables` — современная замена iptables (уже default в Debian 10+, RHEL 8+). Также в облаках часто используют security groups вместо iptables.

---

## systemd и логи

### 18. Как управлять сервисами через systemd?

**Основные команды:**
```bash
# Статус сервиса
systemctl status nginx

# Запустить / остановить / перезапустить
systemctl start nginx
systemctl stop nginx
systemctl restart nginx

# Перезагрузить конфиг без остановки (graceful reload)
systemctl reload nginx

# Включить/отключить автозапуск при старте системы
systemctl enable nginx
systemctl disable nginx

# Включить И сразу запустить
systemctl enable --now nginx

# Перечитать unit-файлы после изменения
systemctl daemon-reload
```

**Просмотр и диагностика:**
```bash
# Список всех запущенных сервисов
systemctl list-units --type=service --state=running

# Сервисы с ошибками
systemctl list-units --type=service --state=failed

# Проверить корректность unit-файла
systemd-analyze verify /etc/systemd/system/myapp.service
```

**Создание собственного unit-файла:**
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal

# Ограничения ресурсов
MemoryMax=512M
CPUQuota=50%
LimitNOFILE=65536

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```

**Типы перезапуска (`Restart=`):**
- `no` — не перезапускать (по умолчанию)
- `on-failure` — только при ненулевом коде возврата
- `always` — всегда (даже при `systemctl stop` — осторожно!)
- `on-abnormal` — при сигналах и таймаутах

---

### 19. Как работать с journalctl? Основные сценарии использования.

`journalctl` — инструмент для чтения журнала systemd. Собирает логи всех сервисов, ядра и системы в структурированный бинарный формат.

```bash
# Все логи (с начала времён, огромный вывод)
journalctl

# Логи конкретного сервиса
journalctl -u nginx
journalctl -u nginx -u postgresql  # несколько сервисов

# Follow (как tail -f)
journalctl -u nginx -f

# Последние N строк
journalctl -u nginx -n 100

# С определённого времени
journalctl -u nginx --since "2024-01-15 10:00:00"
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-15" --until "2024-01-16"

# Только ошибки (приоритеты: emerg, alert, crit, err, warning, notice, info, debug)
journalctl -u nginx -p err
journalctl -p err..warning  # диапазон приоритетов

# Логи текущей загрузки системы
journalctl -b

# Логи предыдущей загрузки (очень полезно при расследовании краша)
journalctl -b -1   # предыдущая загрузка
journalctl -b -2   # позапрошлая

# Логи ядра (аналог dmesg)
journalctl -k
journalctl -k -b -1  # kernel логи прошлой загрузки

# Вывод в JSON (для парсинга)
journalctl -u nginx -o json-pretty | head -50

# Посмотреть занимаемое место
journalctl --disk-usage

# Очистить старые логи
journalctl --vacuum-time=30d   # оставить только за 30 дней
journalctl --vacuum-size=1G    # оставить не более 1 ГБ
```

**Настройка хранения логов** (`/etc/systemd/journald.conf`):
```ini
[Journal]
Storage=persistent     # хранить на диске (persistent/volatile/auto)
Compress=yes
SystemMaxUse=2G        # максимум 2 ГБ на диске
SystemKeepFree=1G      # всегда оставлять 1 ГБ свободным
MaxRetentionSec=1month # хранить не дольше месяца
```

---

### 20. Как определить причину перезагрузки сервера?

Это важная задача при incident response. Есть несколько источников информации.

**1. Когда произошла перезагрузка:**
```bash
# История загрузок
last reboot | head -10

# Или через journalctl
journalctl --list-boots
# Список загрузок с номерами и временем
```

**2. Логи ядра предыдущей загрузки:**
```bash
# Ядро логирует причину shutdown/reboot
journalctl -k -b -1 | tail -50

# Искать явные признаки:
journalctl -k -b -1 | grep -iE "panic|oops|bug|error|reboot|watchdog|oom"
```

**3. Системные логи:**
```bash
# Последние сообщения перед перезагрузкой
journalctl -b -1 | tail -100

# Если использовался rsyslog/syslog
grep -iE "shutdown|reboot|panic|halt" /var/log/syslog
grep -iE "shutdown|reboot|panic|halt" /var/log/messages
```

**Типичные причины и как их распознать:**

| Причина | Признаки |
|---|---|
| OOM Kill | `dmesg \| grep -i oom` — есть записи об убийстве процессов |
| Kernel panic | `journalctl -k -b -1` — сообщение `Kernel panic` в конце |
| Watchdog (hardware timeout) | `journalctl -k \| grep watchdog` |
| Плановое обновление | Запись `shutdown` в `last`, логи apt/yum |
| `reboot` командой | `last reboot` покажет пользователя |
| Питание / hardware | Внезапное прерывание без graceful shutdown в логах |

```bash
# Узнать последний код причины выключения (только для systemd)
journalctl -b -1 -n 5 | grep -i "system is rebooting\|shutdown\|poweroff"

# Проверить аварийный дамп памяти (если настроен kdump)
ls -la /var/crash/
```

---

## Дополнительно

### 21. Как работает /proc filesystem?

`/proc` — это **виртуальная псевдофайловая система** (не существует на диске), создаваемая ядром в памяти. Предоставляет интерфейс для чтения и иногда изменения состояния ядра и процессов.

**Информация о процессах (`/proc/<PID>/`):**
```bash
cat /proc/1234/status       # статус процесса: состояние, память, UID, PID
cat /proc/1234/cmdline      # командная строка (аргументы запуска)
cat /proc/1234/environ      # переменные окружения
cat /proc/1234/fd/          # открытые файловые дескрипторы (symlinks)
ls -la /proc/1234/fd/       # fd/0=stdin, fd/1=stdout, fd/2=stderr
cat /proc/1234/maps         # карта виртуальной памяти
cat /proc/1234/net/tcp      # сетевые соединения процесса
cat /proc/1234/cgroup       # cgroup принадлежность
cat /proc/1234/oom_score    # приоритет OOM killer
```

**Системная информация:**
```bash
cat /proc/cpuinfo    # информация о процессорах
cat /proc/meminfo    # подробная информация о памяти
cat /proc/loadavg    # load average и статистика планировщика
cat /proc/uptime     # время работы системы в секундах
cat /proc/version    # версия ядра
cat /proc/mounts     # смонтированные файловые системы
cat /proc/net/dev    # сетевая статистика по интерфейсам
cat /proc/diskstats  # дисковая статистика (основа для iostat)
```

**Настройка ядра через `/proc/sys/`:**
```bash
# Прочитать параметр
cat /proc/sys/net/ipv4/ip_forward

# Изменить параметр (временно, до перезагрузки)
echo 1 > /proc/sys/net/ipv4/ip_forward

# Постоянные изменения — через sysctl:
sysctl net.ipv4.ip_forward=1
# Или в /etc/sysctl.conf:
# net.ipv4.ip_forward = 1
# После чего: sysctl -p
```

**Практическое применение:**
```bash
# Узнать реальную командную строку процесса (удобнее чем ps)
tr '\0' ' ' < /proc/<PID>/cmdline

# Найти открытые файлы конкретного дескриптора
ls -la /proc/<PID>/fd/1  # куда смотрит stdout

# Проверить переменные окружения запущенного процесса
strings /proc/<PID>/environ | grep PATH
```

---

### 22. Что такое swap и когда его использование — проблема?

**Swap** — дисковое пространство (файл или раздел), которое ядро использует как расширение оперативной памяти. Когда RAM заканчивается, ядро переносит ("вытесняет") редко используемые страницы памяти на диск.

**Текущее состояние:**
```bash
free -h              # строка Swap: total/used/free
swapon --show        # подробности: файл/раздел, размер, приоритет
vmstat 1             # si/so: swap in/out в КБ/с
```

**`vm.swappiness`** — параметр ядра (0-100), определяющий агрессивность использования swap:
```bash
cat /proc/sys/vm/swappiness  # обычно 60 по умолчанию
# 0  — использовать swap только когда RAM полностью исчерпана
# 60 — начинать использовать swap когда RAM заполнена на ~60%
# 100 — активно использовать swap

# Для production-серверов часто снижают:
sysctl vm.vm.swappiness=10
```

**Когда swap — это проблема:**

1. **Активное использование swap (высокое si/so)** — диск в сотни раз медленнее RAM. Если база данных или приложение начинает работать со swap — деградация производительности колоссальная.

2. **Swap используется из-за утечки памяти** — приложение постепенно занимает всю RAM и уходит в swap, а потом система начинает "тормозить".

3. **Swap заполнен + RAM заполнена** — следующий шаг — OOM Killer.

```bash
# Найти процессы, использующие swap
for pid in /proc/[0-9]*/status; do
  comm=$(awk '/^Name/{print $2}' "$pid" 2>/dev/null)
  swap=$(awk '/^VmSwap/{print $2}' "$pid" 2>/dev/null)
  [[ -n "$swap" && "$swap" -gt 0 ]] && echo "$swap kB - $comm"
done | sort -rn | head -10
```

**Swap — не всегда плохо:** наличие swap и его небольшое использование (например, для неактивных страниц) — нормально и даже полезно как safety net. Проблема — когда идёт активный swap I/O.

---

### 23. Как работает sudo и как настраивать /etc/sudoers?

`sudo` позволяет пользователям выполнять команды от имени другого пользователя (по умолчанию `root`), не зная его пароля, согласно правилам в `/etc/sudoers`.

**ВАЖНО:** всегда редактируй sudoers через `visudo` — она проверяет синтаксис перед сохранением. Ошибка в sudoers может полностью заблокировать sudo в системе.

```bash
visudo               # редактировать /etc/sudoers
visudo -f /etc/sudoers.d/myteam  # редактировать отдельный файл
```

**Синтаксис правил:**
```
<кто>  <откуда>=(<от_кого>) <команды>

# Примеры:
# Полный root-доступ для пользователя john
john   ALL=(ALL:ALL) ALL

# Без ввода пароля
john   ALL=(ALL) NOPASSWD: ALL

# Только конкретные команды
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp

# Группа (префикс %)
%sudo  ALL=(ALL:ALL) ALL
%ops   ALL=(root) NOPASSWD: /usr/bin/docker, /usr/bin/kubectl
```

**Подключение дополнительных файлов** (лучшая практика):
```bash
# Вместо правки основного sudoers — создаём отдельные файлы
visudo -f /etc/sudoers.d/deploy-user
```
В `/etc/sudoers` должна быть строка:
```
#includedir /etc/sudoers.d
```

**Логирование sudo-команд:**
```bash
# Все команды, выполненные через sudo
grep sudo /var/log/auth.log
journalctl | grep sudo

# Пример записи:
# john : TTY=pts/0 ; PWD=/home/john ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
```

**Ограничения и best practices:**
- Давай минимально необходимые права (principle of least privilege)
- Избегай `NOPASSWD: ALL` для обычных пользователей
- Используй отдельные файлы в `/etc/sudoers.d/` вместо правки основного файла
- Для CI/CD используй отдельного пользователя с явно ограниченным набором команд
- Регулярно проверяй список пользователей с sudo-правами: `grep -E '^%sudo|^sudo' /etc/group`
