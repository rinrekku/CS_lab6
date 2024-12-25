# Лабораторная работа 6

## Автоматизация переноса, шифрования и резервного копирования каталога с Windows на Linux с использованием SCP

### Задание:

1. **Перенос директории с Windows на Linux каждый день в 18:00, если доступна.**
2. **Шифрование и сжатие перенесённого каталога.**
3. **Хранение последних 2 версий заархивированного каталога в хранилище.**
4. **Резервное копирование хранилища каждый день в 21:00.**


## Ход выполнения:

Для начала был установлен OpenSSH-сервер на Windows, чтобы Linux-машина могла подключаться к ней через SSH для использования SCP. Для упрощения автоматической авторизации я использую метод аутентификации через пару ключей `RSA`. Для этого сгенерируем ключ на сервере и запишем его открытую часть в файл `authorized_keys` в директории `C:/ProgramData/ssh`. Там же в `sshd_config` разрешим аутентификацию через `PubKey` и закроем всю директорию для доступа всем, кроме сисетмы и администраторов, иначе в Windows это приведёт к ошибке запуска сервера.

Созданим простой bash-скрипт, выполняющий следующие действия 

   - Проверка доступности Windows-машины с помощью **ping**.
   - Сжатие и шифрование последней скопированной директории
   - Сохранения её в отдельную папку с ещё одной, предыдущей
   - Копирование с компьютера на Windows новой, последней версии директории


   ```bash
   #!/bin/bash

   WINDOWS_DIR="C:/Users/PC/Documents/\"Ableton Projects\""
   LINUX_DIR="/home/rin/ableton/latest"
   BACKUP_DIR="/home/rin/ableton/legacy"
   BACKUP_FILE="$BACKUP_DIR/backup_$(date +\%Y-\%m-\%d).tar.gz"

   if ping -c 1 192.168.1.240 &> /dev/null; then
     cd $LINUX_DIR; cd ..
     tar -czf - latest | gpg --batch --yes --symmetric --cipher-algo AES256 --passphrase-file /home/rin/.gpg_passphrase -o /home/rin/ableton/legacy/backup_$(date +\%Y-\%m-\%d).tar.gz.gpg
     cd $BACKUP_DIR
     find . -type f -name "backup*.gpg" | sort | head -n -1 | while read file; do
       rm -f "$file"
     done

     rm -rf $LINUX_DIR/*

     scp -r PC@192.168.1.240:"$WINDOWS_DIR" $LINUX_DIR/
     sudo chmod -R u+w $LINUX_DIR
     
   else
     echo "Backup is postponed, the computer is unavailiable."
   fi
   ```

   Для того чтобы скрипт выполнялся каждый день в 18:00, была настроена задача в `cron`. Откроем его от `root`, поскольку следующий этап требует этого пользователя:

   ```bash
   sudo crontab -e
   ```

   Добавим строку:

   ```vim
   0 18 * * * /home/rin/scripts/sync_directory.sh
   ```

   Теперь, предварительно создав директорию в `/mnt`, добавим задание на ещё одно резервное копирование, в этот раз с помощью `rsync`. На данный момент я не имею такой возможности, но в будущем я немерен заменить эту директорию на внешний носитель, такой как `USB`.

   ```bash
   #!/bin/bash

   BACKUP_DIR="/home/rin/ableton"
   BACKUP_STORAGE="/mnt/backup_storage"

   rm -rf $BACKUP_STORAGE/*
   rsync -av --delete $BACKUP_DIR $BACKUP_STORAGE
   ```

   Чтобы резервное копирование выполнялось каждый день в 21:00, была также настроена задача в `cron` `root`:

   ```bash
   0 21 * * * /home/rin/scripts/backup_storage.sh
   ```


### Заключение:

В ходе выполнения лабораторной работы я научился гибко применять все полученные мною в ходе курса информатики знания на практике для собственного удобства и удобства на моём потенциальном рабочем месте, основам работы с утилитами `rsync`, `tar` и `gpg`, планировщиком задач `cron`.

### Ссылки на материалы:

1. [`SCP`](https://man7.org/linux/man-pages/man1/scp.1.html)
2. [`GPG`](https://www.unix.com/man-page/redhat/1/gpg/)
3. [`cron`](https://man7.org/linux/man-pages/man5/crontab.5.html)
4. [`tar`](https://man7.org/linux/man-pages/man1/tar.1.html)
5. [`rsync`](https://man7.org/linux/man-pages/man1/rsync.1.html)

