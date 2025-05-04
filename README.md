# Hardening-Gitlab-server

## Firewall && SSH

Dans un premier temps une configuration du firewall est faite afin de n'autoriser que les port 22 - 80 - 443 et 8060 pour le nginx status, afin de permettre a mon serveur gitlab de fonctionner correctement.
De plus une configuration ssh décente est faite via lynis. 

```
[+] SSH Support
------------------------------------
  - Checking running SSH daemon                               [ FOUND ]
    - Searching SSH configuration                             [ FOUND ]
    - OpenSSH option: AllowTcpForwarding                      [ OK ]
    - OpenSSH option: ClientAliveCountMax                     [ OK ]
    - OpenSSH option: ClientAliveInterval                     [ OK ]
    - OpenSSH option: FingerprintHash                         [ OK ]
    - OpenSSH option: GatewayPorts                            [ OK ]
    - OpenSSH option: IgnoreRhosts                            [ OK ]
    - OpenSSH option: LoginGraceTime                          [ OK ]
    - OpenSSH option: LogLevel                                [ OK ]
    - OpenSSH option: MaxAuthTries                            [ OK ]
    - OpenSSH option: MaxSessions                             [ OK ]
    - OpenSSH option: PermitRootLogin                         [ OK ]
    - OpenSSH option: PermitUserEnvironment                   [ OK ]
    - OpenSSH option: PermitTunnel                            [ OK ]
    - OpenSSH option: Port                                    [ OK ]
    - OpenSSH option: PrintLastLog                            [ OK ]
    - OpenSSH option: StrictModes                             [ OK ]
    - OpenSSH option: TCPKeepAlive                            [ OK ]
    - OpenSSH option: UseDNS                                  [ OK ]
    - OpenSSH option: X11Forwarding                           [ OK ]
    - OpenSSH option: AllowAgentForwarding                    [ OK ]
    - OpenSSH option: AllowUsers                              [ NOT FOUND ]
    - OpenSSH option: AllowGroups                             [ NOT FOUND ]
```
Là dessus une configuration pam va être incluse

A partir de la le git lab est accessible en local :

![image](https://github.com/user-attachments/assets/4fbe357b-2bac-48e6-b33c-f3bf3bfe0a7b)

## Montage du root de GitLab sur un autre volume logique

Je vais utiliser lvm ici :

```
ngermond@debgitlab:/$ sudo lvcreate -L 5G -n gitlab_data gv1
  Logical volume "gitlab_data" created.
ngermond@debgitlab:/$ sudo /sbin/mkfs.ext4 /dev/gv1/gitlab_data
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 1310720 4k blocks and 327680 inodes
Filesystem UUID: 4124dd5e-3fb1-4d9f-85cd-7854fbf42151
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
ngermond@debgitlab:/$ sudo gitlab-ctl stop
ok: down: alertmanager: 0s, normally up
ok: down: gitaly: 0s, normally up
ok: down: gitlab-exporter: 0s, normally up
ok: down: gitlab-kas: 0s, normally up
ok: down: gitlab-workhorse: 0s, normally up
ok: down: logrotate: 1s, normally up
ok: down: nginx: 0s, normally up
ok: down: node-exporter: 1s, normally up
ok: down: postgres-exporter: 0s, normally up
ok: down: postgresql: 1s, normally up
ok: down: prometheus: 0s, normally up
ok: down: puma: 0s, normally up
ok: down: redis: 0s, normally up
ok: down: redis-exporter: 1s, normally up
ok: down: sidekiq: 0s, normally up
```
Ensuite je vais monter le nouveau volume et copier les données
```
root@debgitlab:~# mkdir /var/opt/gitlab
root@debgitlab:~# mount /dev/gv1/gitlab_data /var/opt/gitlab
root@debgitlab:~# rsync -a /var/opt/gitlab.old/ /var/opt/gitlab/
```
On voit du coup que mon /var/opt/gitlab est bien monté au bon endroit : 
```
ngermond@debgitlab:/mnt$ sudo mount | grep /var/opt/gitlab
/dev/mapper/gv1-gitlab_data on /var/opt/gitlab type ext4 (rw,relatime)
```
et si l'on réalise un test :
```
ngermond@debgitlab:/$ sudo touch /var/opt/gitlab/testfile
ngermond@debgitlab:/$ sudo ls -l /var/opt/gitlab/testfile
-rw-r--r-- 1 root root 0 May  3 15:43 /var/opt/gitlab/testfile
ngermond@debgitlab:/$ sudo df /var/opt/gitlab/testfile
Filesystem                  1K-blocks   Used Available Use% Mounted on
/dev/mapper/gv1-gitlab_data   5074592 532924   4263140  12% /var/opt/gitlab
```
On vois que le ficher c'est écrit au bon endroit ! ( avant de le suypprimer évidement pour ne pas laisser un root root )

## Hardening 

Pour ce tp, j'ai travailler sur la partie acces control. J'ai utilisé les fichiers de conf sudoers, sshd_config, faillock.conf, pam et modules + pwd, login.defs.
Je ne vais pas mettre l'entièreté des fichier de conf ici car cela serait trop long. Je vais mettre les points concerné par l'audit lynis.

```
[+] Users, Groups and Authentication
------------------------------------
  - Administrator accounts                                    [ OK ]
  - Unique UIDs                                               [ OK ]
  - Consistency of group files (grpck)                        [ OK ]
  - Unique group IDs                                          [ OK ]
  - Unique group names                                        [ OK ]
  - Password file consistency                                 [ OK ]
  - Password hashing methods                                  [ OK ]
  - Password hashing rounds (minimum)                         [ CONFIGURED ]
  - Query system users (non daemons)                          [ DONE ]
  - NIS+ authentication support                               [ NOT ENABLED ]
  - NIS authentication support                                [ NOT ENABLED ]
  - Sudoers file(s)                                           [ FOUND ]
    - Permissions for directory: /etc/sudoers.d               [ OK ]
    - Permissions for: /etc/sudoers                           [ OK ]
    - Permissions for: /etc/sudoers.d/README                  [ OK ]
  - PAM password strength tools                               [ OK ]
  - PAM configuration files (pam.conf)                        [ FOUND ]
  - PAM configuration files (pam.d)                           [ FOUND ]
  - PAM modules                                               [ FOUND ]
  - LDAP module in PAM                                        [ NOT FOUND ]
  - Accounts without expire date                              [ SUGGESTION ]
  - Accounts without password                                 [ OK ]
  - Locked accounts                                           [ OK ]
  - User password aging (minimum)                             [ CONFIGURED ]
  - User password aging (maximum)                             [ CONFIGURED ]
  - Checking expired passwords                                [ OK ]
  - Checking Linux single user mode authentication            [ OK ]
  - Determining default umask
    - umask (/etc/profile)                                    [ OK ]
    - umask (/etc/login.defs)                                 [ OK ]
  - LDAP authentication support                               [ NOT ENABLED ]
  - Logging failed login attempts                             [ ENABLED ]
```
```
[+] File Permissions
------------------------------------
  - Starting file permissions check
    File: /boot/grub/grub.cfg                                 [ OK ]
    File: /etc/crontab                                        [ OK ]
    File: /etc/group                                          [ OK ]
    File: /etc/group-                                         [ OK ]
    File: /etc/hosts.allow                                    [ OK ]
    File: /etc/hosts.deny                                     [ OK ]
    File: /etc/issue                                          [ OK ]
    File: /etc/issue.net                                      [ OK ]
    File: /etc/motd                                           [ OK ]
    File: /etc/passwd                                         [ OK ]
    File: /etc/passwd-                                        [ OK ]
    File: /etc/ssh/sshd_config                                [ OK ]
    Directory: /root/.ssh                                     [ OK ]
```
