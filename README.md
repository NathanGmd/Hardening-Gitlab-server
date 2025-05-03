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
