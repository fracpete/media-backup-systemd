# media-backup-systemd
systemd-based backup of Raspberry Pi media server (*mediaserver.lan*) to backup server (*backupserver.lan*) using `rsync`.

Consists of two timer-based systemd jobs:
* **media_permissions** - fixes the permissions on the media server to ensure that the backup can work properly (runs every day at 1am)
* **media_backup** - uses rsync to sync media folder on media server with one on the backup server (runs every day at 2am)

## Backup user

Create a user `pibak` for handling the backup operation.

**NB:** Adjust numeric group/user IDs as appropriate.

### Media server

* create group **pibak**: `groupadd -g 1002 pibak`
* create user **pibak**: `useradd pibak -u 1001 -g 1002 -m -s /bin/bash`
* log in as user **pibak**
* create ssh key (no passphrase): `ssh-keygen`
* output public key: `cat ~/.ssh/id_rsa.pub`

### Backup server

* create group **pibak**: `groupadd -g 1002 pibak`
* create user **pibak**: `useradd pibak -u 1001 -g 1002 -m -s /bin/bash`
* log in as user **pibak**
* paste the media-server public key into: `~/.ssh/authorized_keys`

## Scripts (media server)

### Fixing permissions

```bash
#!/bin/bash

chown pi:users /media/xbmc/media
find /media/xbmc/media -type d -exec chmod 775 {} \;
```

### Syncing media files

```bash
#!/bin/bash

timeout 120m rsync -av --rsh=ssh --modify-window=3601 /media/xbmc/media/ backupserver.lan:/media/backup/media/xbmc
```

## Systemd (media server)

* located in `/etc/systemd/system`
* media_permissions.service
  ```
  [Unit]
  Description="Fixes the permissions of the media files and directories."
  
  [Service]
  ExecStart=/usr/local/bin/media_permissions.sh
  ```
* media_permissions.timer
  ```
  [Unit]
  Description="Run media_permissions.service every night at 1am"
  
  [Timer]
  OnCalendar=Mon..Sun *-*-* 01:00:*
  Unit=media_permissions.service
  
  [Install]
  WantedBy=multi-user.target
  ```
* media_backup.service
  ```
  [Unit]
  Description="Backs up the media to backupserver.lan"
  
  [Service]
  User=pibak
  Group=pibak
  ExecStart=/usr/local/bin/media_backup.sh
  ```
* media_backup.timer
  ```
  [Unit]
  Description="Run media_backup.service every night at 2am"
  
  [Timer]
  OnCalendar=Mon..Sun *-*-* 02:00:*
  Unit=media_backup.service
  
  [Install]
  WantedBy=multi-user.target
  ```
