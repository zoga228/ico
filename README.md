# CTF Playbook: WSL2 + Ubuntu

Практическая инструкция по решению CTF-задач категорий Web, Crypto, PWN, Reverse, Forensics и Misc с WSL2/Ubuntu и ИИ-аналитиком.

> Используй команды только для собственных файлов, локальных лабораторий и явно разрешённых CTF-сервисов. Не сканируй чужие системы.

## 1. Окружение

- WSL2 + Ubuntu 26.04.1 LTS
- пользователь: zoga
- Python 3.14.4
- рабочая директория: ~/ctf
- venv: ~/ctf/venv

После открытия Ubuntu:

    source ~/ctf/activate.sh
    python --version
    which python
    which gdb
    which jadx

Подготовлены:

- диагностика: file, strings, xxd, sha256sum, readelf, objdump
- сеть/Web: curl, wget, HTTPie, nmap, tshark, Burp Suite Community
- reverse: Ghidra, radare2, JADX, apktool, GDB, pwndbg, Ropper
- PWN: pwntools, GDB/pwndbg, Ropper
- crypto: Python, PyCryptodome, SymPy, Z3, OpenSSL
- forensics: binwalk, ExifTool, foremost, tcpdump, TShark, YARA, Volatility 3
- разработка: GCC, Git, Make, Docker Engine, Docker Compose, Python venv

Metasploit, Kali Linux, Nessus, OpenVAS, cracked Burp, cracked IDA и exploit packs не нужны для этого базового workflow и не устанавливались.

## 2. Универсальный workflow

Для каждой задачи создавай отдельную директорию:

    mkdir -p ~/ctf/<category>/<challenge-name>
    cd ~/ctf/<category>/<challenge-name>

Оригиналы не изменяй:

    mkdir -p original work output notes
    cp /path/to/challenge ./original/
    cp -a ./original/. ./work/
    sha256sum ./original/* | tee notes/hashes.txt
    find . -maxdepth 2 -type f -printf '%p\n' | sort
    file ./work/*

Цикл решения:

1. собрать факты;
2. сформулировать до трёх гипотез;
3. выполнить одну небольшую проверку;
4. сохранить полный вывод;
5. обновить гипотезу;
6. написать solver/decode/exploit;
7. повторить решение на чистой копии.

Сохраняй выводы:

    command 2>&1 | tee notes/01-command.txt

## 3. Как использовать ИИ

ИИ должен быть аналитиком, а не генератором непроверенных догадок. Передавай ему категорию, цель, рабочую директорию, команду, полный вывод и уже проверенные факты. Не отправляй целиком бинарники и огромные base64-дампы, если достаточно извлечь нужный фрагмент.

### Универсальный prompt

    Ты мой CTF-аналитик и наставник.

    Категория: [web/crypto/pwn/reverse/forensics/misc]
    Название: [challenge]
    Цель: [найти flag / понять уязвимость / восстановить алгоритм]
    Разрешённая область: только локальные файлы и явно указанный CTF-сервис.
    Окружение: WSL2 Ubuntu, Python 3.14, инструменты из ~/ctf.
    Рабочая директория: [абсолютный путь]
    Уже проверено: [факты]
    Не делай: не сканируй внешние цели, не удаляй файлы, не устанавливай пакеты без согласия, не используй curl|sh, не выдумывай результаты.

    Формат каждого ответа:
    1. Наблюдения — только факты из вывода.
    2. Гипотезы — максимум 3 и уверенность.
    3. Следующая команда — одна конкретная безопасная проверка.
    4. Что ожидаем увидеть.
    5. Как трактовать варианты результата.
    6. Если нужен код — минимальный скрипт с входом и выходом.

    Сначала проведи triage. Не переходи к brute force или exploit,
    пока не объяснишь, почему это оправдано.

Результаты передавай так:

    Команда:
    file ./work/chall

    Вывод:
    [полный вывод]

    Вопрос:
    Какие две следующие проверки дадут максимум информации?

После шага проси вести журнал:

    Обнови журнал:
    - подтверждено;
    - опровергнуто;
    - неизвестно;
    - следующая проверка;
    - вероятности гипотез.

Большой контекст 100K полезен для исходников, декомпиляции и нескольких логов, но не заменяет структуру. Лучше одна гипотеза и одна проверка за итерацию.

## 4. Reverse Engineering

Быстрый triage ELF:

    cd ~/ctf/reverse/<challenge-name>
    file ./chall
    sha256sum ./chall
    checksec --file=./chall 2>&1
    strings -a -n 4 ./chall | tee notes/strings.txt
    readelf -h ./chall
    readelf -S ./chall
    readelf -s ./chall | head -100
    rabin2 -I ./chall
    rabin2 -zz ./chall

Android:

    file app.apk
    unzip -l app.apk | head -100
    jadx -d jadx-out app.apk
    apktool d -o apktool-out app.apk

Динамический анализ:

    ltrace ./chall
    strace -f -o notes/strace.txt ./chall
    gdb -q ./chall

Минимум GDB:

    set pagination off
    set disassembly-flavor intel
    info functions
    info files
    info registers
    disassemble main
    break main
    run

radare2:

    r2 -A ./chall
    aaa
    afl
    pdf @ main
    iz

Порядок: найти ввод, найти сравнение с ожидаемым значением, восстановить преобразование, написать обратный скрипт, проверить результат.

Prompt: «По выводу file/readelf/strings и фрагменту декомпиляции определи архитектуру, защитные механизмы, точку ввода, функции проверки и одну следующую проверку. Не угадывай алгоритм по имени функции».

## 5. Crypto

Сначала классифицируй задачу: encoding, hash, XOR, AES/RSA/ECC, PRNG, слабая реализация или математическая задача.

    file challenge.txt
    wc -c challenge.txt
    xxd -g 1 -l 256 challenge.txt
    strings -a challenge.txt | head -100

Проверь длины, повторения блоков, известный plaintext, формат flag, повтор nonce/IV, слабый seed, малые RSA-параметры, общие множители и утечки битов.

В Python доступны PyCryptodome, SymPy и Z3:

    from Crypto.Cipher import AES
    from Crypto.Util.number import long_to_bytes, bytes_to_long

Не начинай с полного brute force. Сначала попроси ИИ составить таблицу известных значений, неизвестных, ограничений и проверок. Для solver сначала получи математическую модель, затем код.

Prompt: «Не называй алгоритм только по внешнему виду. Составь таблицу длин, повторений и ограничений; предложи максимум три гипотезы и по одной информативной проверке для каждой».

## 6. Web

Работай только с локальной задачей или разрешённым сервисом.

Локальная Docker-лаборатория:

    cd ~/ctf/docker/web-lab
    sudo docker compose up -d --build
    curl -i http://127.0.0.1:5000
    sudo docker compose down

Пассивный triage:

    curl -i http://127.0.0.1:5000/
    curl -sS -D notes/headers.txt -o notes/body.html http://127.0.0.1:5000/
    http -v GET http://127.0.0.1:5000/

Проверь cookies и флаги HttpOnly/Secure/SameSite, redirect, методы, формы, JSON, скрытые поля, robots.txt, sitemap.xml, различия ответов, авторизацию и контроль доступа.

Burp используй для просмотра и повторения запросов. ffuf — только на разрешённой цели и с маленькой скоростью:

    ffuf -u http://127.0.0.1:5000/FUZZ -w ./wordlist.txt -rate 20 -mc all

Prompt: «Это локальная/разрешённая Web CTF-задача. Проанализируй запрос и ответ по маршрутизации, cookies/session, входным параметрам и контролю доступа. Дай одну безопасную следующую проверку. Не предлагай широкое сканирование».

## 7. PWN

Начало:

    file ./chall
    checksec --file=./chall
    ldd ./chall
    strings -a ./chall | head -100
    gdb -q ./chall

Минимальный pwntools-каркас:

    from pwn import *
    context.binary = elf = ELF('./chall', checksec=False)
    io = process(elf.path)
    io.sendlineafter(b'> ', b'test')
    print(io.recvall(timeout=1))

Для offset:

    from pwn import *
    print(cyclic(300))
    print(cyclic_find(0x6161616c))

Гаджеты:

    ropper --file ./chall --search 'pop rdi; ret'

Порядок: crash и точный offset, контролируемые регистры/адреса, утечка и база PIE/libc, выбор ret2win/ret2libc/ROP/format string/shellcode, локальная проверка exploit.

## 8. Forensics

Работай с копией:

    mkdir -p original work extracted notes
    cp evidence.bin original/
    cp evidence.bin work/
    sha256sum original/evidence.bin | tee notes/hash.txt

Базовый triage:

    file work/evidence.bin
    exiftool work/evidence.bin
    strings -a -n 6 work/evidence.bin | tee notes/strings.txt
    xxd -g 1 -l 256 work/evidence.bin
    binwalk work/evidence.bin
    foremost -i work/evidence.bin -o extracted

Архивы:

    unzip -l work/file.zip
    7z l work/file.7z

PCAP:

    capinfos capture.pcap
    tshark -r capture.pcap -q -z io,phs
    tshark -r capture.pcap -Y 'http' -T fields -e frame.number -e ip.src -e ip.dst -e http.request.uri

Память:

    vol -f memory.raw windows.info
    vol -f memory.raw windows.pslist
    vol -f memory.raw windows.netscan

Prompt: «Я работаю с копией. По file, hash, exiftool, strings и первым байтам определи тип, подозрительные признаки, вероятные вложенные данные и одну следующую проверку. Не изменяй оригинал».

## 9. Misc

Сначала отнеси задачу к encoding, текстовому формату, логике, автоматизации, протоколу, steganography или комбинации.

    file artifact
    sha256sum artifact
    strings -a artifact | head -100
    xxd -g 1 -l 256 artifact

Выбирай узкий эксперимент, а не десятки декодеров вслепую.

## 10. Структура задачи и заметки

    challenge-name/
    ├── original/
    ├── work/
    ├── output/
    ├── notes/
    │   ├── 00-scope.txt
    │   ├── 01-triage.txt
    │   ├── hashes.txt
    │   └── hypotheses.md
    ├── solve.py
    ├── exploit.py
    └── README.md

Шаблон hypotheses.md:

    Гипотеза:
    Факты за:
    Факты против:
    Следующая проверка:
    Результат:
    Статус: open / confirmed / rejected

Если вывод огромный:

    command 2>&1 | tee notes/full.txt
    sed -n '1,160p' notes/full.txt
    rg -n 'flag|pass|check|key|secret|admin|token|strcmp|memcmp' notes/full.txt

## 11. Перед сдачей

- повтори решение с чистой копии;
- убери случайные абсолютные пути;
- проверь зависимости;
- сохрани solver/exploit;
- опиши найденный механизм;
- не публикуй реальные секреты, токены, cookies и приватные данные.

Финальный отчёт:

    Категория:
    Уязвимость/идея:
    Входные данные:
    Ключевая команда или код:
    Почему это работает:
    Полученный результат:
    Как воспроизвести с чистого состояния:
