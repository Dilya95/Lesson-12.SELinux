# Домашнее задание 15: Практика c SELinux

## Задания
Описание домашнего задания
1. Запустить Nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/Nickmob/vagrant_selinux_dns_problems; 
выяснить причину неработоспособности механизма обновления зоны (см. README);
предложить решение (или решения) для данной проблемы;
выбрать одно из решений для реализации, предварительно обосновав выбор;
реализовать выбранное решение и продемонстрировать его работоспособность.


## Выполнение
### Запустить Nginx на нестандартном порту 3-мя разными способами

### Изменила порт Nginx на нестандартный. Перезапустила Nginx, словила ошибку, посмотрела логи.
```
[root@otus-homework ~]# sudo sed -i 's/listen       80;/listen       8218;/' /etc/nginx/nginx.conf

[root@otus-homework ~]# grep -n "listen" /etc/nginx/nginx.conf
38:        listen       8218;
39:        listen       [::]:80;
50:#        listen       443 ssl;
51:#        listen       [::]:443 ssl;

[root@otus-homework ~]# sudo systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

[root@otus-homework ~]# systemctl status nginx -l
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Mon 2026-06-01 14:06:49 UTC; 7s ago
   Duration: 14min 30.217s
 Invocation: 7973019074f946919c82ddc68d6524e6
    Process: 1826 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1828 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
   Mem peak: 2.9M
        CPU: 36ms

Jun 01 14:06:49 otus-homework systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jun 01 14:06:49 otus-homework nginx[1828]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 01 14:06:49 otus-homework nginx[1828]: nginx: [emerg] bind() to 0.0.0.0:8218 failed (13: Permission denied)
Jun 01 14:06:49 otus-homework nginx[1828]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 01 14:06:49 otus-homework systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Jun 01 14:06:49 otus-homework systemd[1]: nginx.service: Failed with result 'exit-code'.
Jun 01 14:06:49 otus-homework systemd[1]: Failed to start nginx.service - The nginx HTTP and reverse proxy server.




[root@otus-homework ~]# sudo sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 8218.

*****  Plugin bind_ports (92.2 confidence) suggests   ************************

If you want to allow /usr/sbin/nginx to bind to network port 8218
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 8218
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Plugin catchall_boolean (7.83 confidence) suggests   ******************

If you want to allow nis to enabled
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1

*****  Plugin catchall (1.41 confidence) suggests   **************************

If you believe that nginx should be allowed name_bind access on the port 8218 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -X 300 -i my-nginx.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:unreserved_port_t:s0
Target Objects                port 8218 [ tcp_socket ]
Source                        nginx
Source Path                   /usr/sbin/nginx
Port                          8218
Host                          <Unknown>
Source RPM Packages           nginx-core-1.26.3-6.el10_2.3.x86_64
Target RPM Packages           
SELinux Policy RPM            selinux-policy-targeted-42.1.7-1.el10.noarch
Local Policy RPM              selinux-policy-targeted-42.1.7-1.el10.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     otus-homework
Platform                      Linux otus-homework 6.12.0-124.8.1.el10_1.x86_64
                              #1 SMP PREEMPT_DYNAMIC Tue Nov 11 11:41:04 EST
                              2025 x86_64
Alert Count                   1
First Seen                    2026-06-01 14:06:49 UTC
Last Seen                     2026-06-01 14:06:49 UTC
Local ID                      ca52d29c-a525-4cf1-ba2d-8c0f2f1c4aeb

Raw Audit Messages
type=AVC msg=audit(1780322809.299:282): avc:  denied  { name_bind } for  pid=1828 comm="nginx" src=8218 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1780322809.299:282): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=55cb04172bf8 a2=10 a3=7ffdb9f6b4c0 items=0 ppid=1 pid=1828 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=unset UID=root GID=root EUID=root SUID=root FSUID=root EGID=root SGID=root FSGID=root

Hash: nginx,httpd_t,unreserved_port_t,tcp_socket,name_bind


[root@otus-homework ~]# sudo ausearch -c 'nginx' --raw
type=AVC msg=audit(1780322809.299:282): avc:  denied  { name_bind } for  pid=1828 comm="nginx" src=8218 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1780322809.299:282): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55cb04172bf8 a2=10 a3=7ffdb9f6b4c0 items=0 ppid=1 pid=1828 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=PROCTITLE msg=audit(1780322809.299:282): proctitle=2F7573722F7362696E2F6E67696E78002D74
```

### Использование переключателей setsebool
```
[root@otus-homework ~]# getsebool nis_enabled
nis_enabled --> off

[root@otus-homework ~]# setsebool -P nis_enabled on

[root@otus-homework ~]# getsebool nis_enabled
nis_enabled --> on

[root@otus-homework ~]# systemctl restart nginx

[root@otus-homework ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Mon 2026-06-01 14:11:46 UTC; 32ms ago
 Invocation: bf74ab456a1946068d92d6a7c4e215b4
    Process: 1877 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1879 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1881 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1882 (nginx)
      Tasks: 3 (limit: 24636)
     Memory: 3.1M (peak: 3.1M)
        CPU: 54ms
     CGroup: /system.slice/nginx.service
             ├─1882 "nginx: master process /usr/sbin/nginx"
             ├─1883 "nginx: worker process"
             └─1884 "nginx: worker process"

Jun 01 14:11:46 otus-homework systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jun 01 14:11:46 otus-homework nginx[1879]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 01 14:11:46 otus-homework nginx[1879]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 01 14:11:46 otus-homework systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.


[root@otus-homework ~]# curl http://185.147.26.97:8218
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Mon, 01 Jun 2026 14:15:09 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Thu, 28 Nov 2024 18:02:45 GMT
Connection: keep-alive
ETag: "6748b045-1680"
Accept-Ranges: bytes

Потом возвраащем все обратно:
[root@otus-homework ~]# sudo setsebool -P nis_enabled off

[root@otus-homework ~]# sudo systemctl stop nginx

[root@otus-homework ~]# getsebool nis_enabled
nis_enabled --> off

```
Недостаток решения: boolean nis_enabled расширяет разрешения шире, чем нужно только для nginx. Поэтому для production это решение менее предпочтительно, чем точечно добавить порт в http_port_t.


### Добавление нестандартного порта в имеющийся тип
```
[root@otus-homework ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
pegasus_http_port_t            tcp      5988

[root@otus-homework ~]# semanage port -a -t http_port_t -p tcp 8218

[root@otus-homework ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      8218, 80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
pegasus_http_port_t            tcp      5988

[root@otus-homework ~]# sudo systemctl restart nginx

[root@otus-homework ~]# sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Mon 2026-06-01 14:19:23 UTC; 7s ago
 Invocation: a872cdc937d34b2bb3455ca3fc0eb53e
    Process: 1947 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1949 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1950 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1952 (nginx)
      Tasks: 3 (limit: 24636)
     Memory: 3.1M (peak: 3.1M)
        CPU: 57ms
     CGroup: /system.slice/nginx.service
             ├─1952 "nginx: master process /usr/sbin/nginx"
             ├─1953 "nginx: worker process"
             └─1954 "nginx: worker process"

Jun 01 14:19:23 otus-homework systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jun 01 14:19:23 otus-homework nginx[1949]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 01 14:19:23 otus-homework nginx[1949]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 01 14:19:23 otus-homework systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.



[root@otus-homework ~]# curl http://185.147.26.97:8218 -i 
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Mon, 01 Jun 2026 14:19:40 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Thu, 28 Nov 2024 18:02:45 GMT
Connection: keep-alive
ETag: "6748b045-1680"
Accept-Ranges: bytes

Потом возвращаем все обратно:
[root@otus-homework ~]# semanage port -d -t http_port_t -p tcp 8218

[root@otus-homework ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
pegasus_http_port_t            tcp      5988

[root@otus-homework ~]# sudo systemctl stop nginx


```
Это наиболее точное решение для случая, когда нужно разрешить nginx слушать конкретный нестандартный HTTP-порт.



### Формирование и установка модуля SELinux.
```
[root@otus-homework ~]# bash -c 'echo > /var/log/audit/audit.log'

[root@otus-homework ~]# sudo systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

[root@otus-homework ~]# sudo ausearch -c 'nginx' --raw
type=AVC msg=audit(1780323768.997:372): avc:  denied  { name_bind } for  pid=1998 comm="nginx" src=8218 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1780323768.997:372): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55cee665ac48 a2=10 a3=7ffdca6ee140 items=0 ppid=1 pid=1998 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=PROCTITLE msg=audit(1780323768.997:372): proctitle=2F7573722F7362696E2F6E67696E78002D74

[root@otus-homework ~]# sudo ausearch -c 'nginx' --raw | audit2why
type=AVC msg=audit(1780323768.997:372): avc:  denied  { name_bind } for  pid=1998 comm="nginx" src=8218 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

     Was caused by:
     The boolean nis_enabled was set incorrectly. 
     Description:
     Allow nis to enabled

     Allow access by executing:
     # setsebool -P nis_enabled 1

[root@otus-homework ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@otus-homework ~]# cat my-nginx.pp
??|???|?SE Linux Modumy-nginx1.0@
tcp_socket     name_binobject_r@@@@@httpd_t@unreserved_port_t@@@@@@@@@@@@@@@@@@@@@@@@@@

[root@otus-homework ~]# semodule -i my-nginx.pp

[root@otus-homework ~]# semodule -l | grep nginx
my-nginx

[root@otus-homework ~]# sudo systemctl restart nginx

[root@otus-homework ~]# sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Mon 2026-06-01 14:24:22 UTC; 4s ago
 Invocation: 85aa95a5afbc472480462bd1c3734599
    Process: 2031 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 2033 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 2035 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 2036 (nginx)
      Tasks: 3 (limit: 24636)
     Memory: 3.1M (peak: 3.2M)
        CPU: 61ms
     CGroup: /system.slice/nginx.service
             ├─2036 "nginx: master process /usr/sbin/nginx"
             ├─2037 "nginx: worker process"
             └─2038 "nginx: worker process"

Jun 01 14:24:22 otus-homework systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jun 01 14:24:22 otus-homework nginx[2033]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 01 14:24:22 otus-homework nginx[2033]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 01 14:24:22 otus-homework systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.


[root@otus-homework ~]# curl http://185.147.26.97:8218 -i 
HTTP/1.1 200 OK
Server: nginx/1.26.3
Date: Mon, 01 Jun 2026 14:24:44 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Thu, 28 Nov 2024 18:02:45 GMT
Connection: keep-alive
ETag: "6748b045-1680"
Accept-Ranges: bytes


```
Недостаток решения: audit2allow может создать слишком широкие разрешения, если не анализировать содержимое .te-файла. Поэтому такой способ стоит применять только после понимания причины отказа.




### Обеспечить работоспособность приложения при включенном SELinux.
### Клоним себе и разворачиваем вмки:
```
dilyam@MacBook-Pro-Dilya-2 Ansible % git clone https://github.com/Nickmob/vagrant_selinux_dns_problems.git
Клонирование в «vagrant_selinux_dns_problems»...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 32 (delta 9), reused 29 (delta 9), pack-reused 0 (from 0)
Получение объектов: 100% (32/32), 7.23 КиБ | 2.41 МиБ/с, готово.
Определение изменений: 100% (9/9), готово.

dilyam@MacBook-Pro-Dilya-2 Ansible % cd vagrant_selinux_dns_problems

dilyam@MacBook-Pro-Dilya-2 vagrant_selinux_dns_problems % vagrant up --provider=vmware_desktop
Bringing machine 'ns01' up with 'vmware_desktop' provider...
Bringing machine 'client' up with 'vmware_desktop' provider...
==> ns01: Box 'almalinux/9' could not be found. Attempting to find and install...
    ns01: Box Provider: vmware_desktop, vmware_fusion, vmware_workstation
    ns01: Box Version: 9.4.20240805
==> ns01: Loading metadata for box 'almalinux/9'
    ns01: URL: https://vagrantcloud.com/api/v2/vagrant/almalinux/9
==> ns01: Adding box 'almalinux/9' (v9.4.20240805) for provider: vmware_desktop (arm64)
    ns01: Downloading: https://vagrantcloud.com/almalinux/boxes/9/versions/9.4.20240805/providers/vmware_desktop/arm64/vagrant.box
    ns01: Calculating and comparing box checksum...
==> ns01: Successfully added box 'almalinux/9' (v9.4.20240805) for 'vmware_desktop (arm64)'!
==> ns01: Cloning VMware VM: 'almalinux/9'. This can take some time...
==> ns01: Checking if box 'almalinux/9' version '9.4.20240805' is up to date...
==> ns01: Verifying vmnet devices are healthy...
==> ns01: Preparing network adapters...
==> ns01: Fixed port collision for 22 => 2222. Now on port 2200.
==> ns01: Starting the VMware VM...
==> ns01: Waiting for the VM to receive an address...
==> ns01: Forwarding ports...
    ns01: -- 22 => 2200
==> ns01: Waiting for machine to boot. This may take a few minutes...
    ns01: SSH address: 127.0.0.1:2200
    ns01: SSH username: vagrant
    ns01: SSH auth method: private key
    ns01: 
    ns01: Vagrant insecure key detected. Vagrant will automatically replace
    ns01: this with a newly generated keypair for better security.
    ns01: 
    ns01: Inserting generated public key within guest...
    ns01: Removing insecure key from the guest if it's present...
    ns01: Key inserted! Disconnecting and reconnecting using new SSH key...
==> ns01: Machine booted and ready!
==> ns01: Setting hostname...
==> ns01: Configuring network adapters within the VM...
==> ns01: Running provisioner: ansible...
Vagrant gathered an unknown Ansible version:

and falls back on the compatibility mode '1.8'.

Alternatively, the compatibility mode can be specified in your Vagrantfile:
https://www.vagrantup.com/docs/provisioning/ansible_common.html#compatibility_mode

    ns01: Running ansible-playbook...

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [ns01]

TASK [install packages] ********************************************************
changed: [ns01]

PLAY [ns01] ********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [ns01]

TASK [copy named.conf] *********************************************************
changed: [ns01]

TASK [copy master zone dns.lab] ************************************************
changed: [ns01] => (item=/Users/dilyam/Ansible/vagrant_selinux_dns_problems/provisioning/files/ns01/named.dns.lab)
changed: [ns01] => (item=/Users/dilyam/Ansible/vagrant_selinux_dns_problems/provisioning/files/ns01/named.dns.lab.view1)

TASK [copy dynamic zone ddns.lab] **********************************************
changed: [ns01]

TASK [copy dynamic zone ddns.lab.view1] ****************************************
changed: [ns01]

TASK [copy master zone newdns.lab] *********************************************
changed: [ns01]

TASK [copy rev zones] **********************************************************
changed: [ns01]

TASK [copy resolv.conf to server] **********************************************
changed: [ns01]

TASK [copy transferkey to server] **********************************************
changed: [ns01]

TASK [set /etc/named permissions] **********************************************
changed: [ns01]

TASK [set /etc/named/dynamic permissions] **************************************
changed: [ns01]

TASK [ensure named is running and enabled] *************************************
changed: [ns01]
[WARNING]: Could not match supplied host pattern, ignoring: client

PLAY [client] ******************************************************************
skipping: no hosts matched

PLAY RECAP *********************************************************************
ns01                       : ok=14   changed=12   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

==> client: Box 'almalinux/9' could not be found. Attempting to find and install...
    client: Box Provider: vmware_desktop, vmware_fusion, vmware_workstation
    client: Box Version: 9.4.20240805
==> client: Loading metadata for box 'almalinux/9'
    client: URL: https://vagrantcloud.com/api/v2/vagrant/almalinux/9
==> client: Adding box 'almalinux/9' (v9.4.20240805) for provider: vmware_desktop (arm64)
==> client: Cloning VMware VM: 'almalinux/9'. This can take some time...
==> client: Checking if box 'almalinux/9' version '9.4.20240805' is up to date...
==> client: Verifying vmnet devices are healthy...
==> client: Preparing network adapters...
==> client: Fixed port collision for 22 => 2222. Now on port 2201.
==> client: Starting the VMware VM...
==> client: Waiting for the VM to receive an address...
==> client: Forwarding ports...
    client: -- 22 => 2201
==> client: Waiting for machine to boot. This may take a few minutes...
    client: SSH address: 127.0.0.1:2201
    client: SSH username: vagrant
    client: SSH auth method: private key
    client: 
    client: Vagrant insecure key detected. Vagrant will automatically replace
    client: this with a newly generated keypair for better security.
    client: 
    client: Inserting generated public key within guest...
    client: Removing insecure key from the guest if it's present...
    client: Key inserted! Disconnecting and reconnecting using new SSH key...
==> client: Machine booted and ready!
==> client: Setting hostname...
==> client: Configuring network adapters within the VM...
==> client: Running provisioner: ansible...
Vagrant gathered an unknown Ansible version:

and falls back on the compatibility mode '1.8'.

Alternatively, the compatibility mode can be specified in your Vagrantfile:
https://www.vagrantup.com/docs/provisioning/ansible_common.html#compatibility_mode

    client: Running ansible-playbook...

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [client]

TASK [install packages] ********************************************************
changed: [client]

PLAY [ns01] ********************************************************************
skipping: no hosts matched

PLAY [client] ******************************************************************

TASK [Gathering Facts] *********************************************************
ok: [client]

TASK [copy resolv.conf to the client] ******************************************
changed: [client]

TASK [copy rndc conf file] *****************************************************
changed: [client]

TASK [copy motd to the client] *************************************************
changed: [client]

TASK [copy transferkey to client] **********************************************
changed: [client]

PLAY RECAP *********************************************************************
client                     : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


Проверяем статус этих ВМ:
dilyam@MacBook-Pro-Dilya-2 vagrant_selinux_dns_problems % vagrant status
Current machine states:

ns01                      running (vmware_desktop)
client                    running (vmware_desktop)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

### Подключаемся к ВМ client и попробуем внести изменения в зону:
```
dilyam@MacBook-Pro-Dilya-2 vagrant_selinux_dns_problems % vagrant ssh client
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
Last login: Tue Jun  2 13:08:45 2026 from 172.16.191.1

[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1780405620.683:705): avc:  denied  { dac_read_search } for  pid=3469 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

    Was caused by:
        Missing type enforcement (TE) allow rule.

        You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1780405620.683:705): avc:  denied  { dac_override } for  pid=3469 comm="20-chrony-dhcp" capability=1  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

    Was caused by:
        Missing type enforcement (TE) allow rule.

        You can use audit2allow to generate a loadable module to allow this access.


[root@client ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 May 21 10:22 /var/named/named.localhost


[root@client ~]# ls -laZ /etc/named
total 12
drwxr-x---.  2 root named system_u:object_r:named_conf_t:s0    6 May 21 10:30 .
drwxr-xr-x. 84 root root  system_u:object_r:etc_t:s0        8192 Jun  2 13:08 ..

```


### Подключаемся к ВМ ns01:
```
dilyam@MacBook-Pro-Dilya-2 vagrant_selinux_dns_problems % vagrant ssh ns01  
Last login: Tue Jun  2 13:06:01 2026 from 172.16.191.1
[vagrant@ns01 ~]$ sudo -i

В логах мы видим, что ошибка в контексте безопасности. Целевой контекст named_conf_t.
Для сравнения посмотрим существующую зону (localhost) и её контекст:

[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1780405449.502:705): avc:  denied  { dac_read_search } for  pid=3473 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

    Was caused by:
        Missing type enforcement (TE) allow rule.

        You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1780405449.502:705): avc:  denied  { dac_override } for  pid=3473 comm="20-chrony-dhcp" capability=1  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

    Was caused by:
        Missing type enforcement (TE) allow rule.

        You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1780406038.046:1799): avc:  denied  { write } for  pid=8643 comm="isc-net-0001" name="dynamic" dev="nvme0n1p3" ino=17008814 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

    Was caused by:
        Missing type enforcement (TE) allow rule.

        You can use audit2allow to generate a loadable module to allow this access.


[root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 May 21 10:22 /var/named/named.localhost


[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_conf_t:s0      121 Jun  2 13:06 .
drwxr-xr-x. 84 root root  system_u:object_r:etc_t:s0            8192 Jun  2 13:06 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_conf_t:s0   56 Jun  2 13:05 dynamic
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      784 Jun  2 13:05 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      610 Jun  2 13:05 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      609 Jun  2 13:05 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      657 Jun  2 13:05 named.newdns.lab

Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: 

[root@ns01 ~]# sudo semanage fcontext -l | grep named
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0 
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0 
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0 
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
/etc/named(/.*)?                                   all files          system_u:object_r:named_conf_t:s0 
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0 
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0 
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0 
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0 
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0 
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0 
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0 
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0 
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0 
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0 
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0 
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0 
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0 
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0 
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0 
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0 
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0 
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0 
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0 
/var/named/chroot/dev/urandom                      character device   system_u:object_r:urandom_device_t:s0 
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0 
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0 
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0 
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0 
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0 
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0 
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0 
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0 
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0 
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0 
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0 
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0 
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0 
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0 
/var/named/chroot_sdb/dev/urandom                  character device   system_u:object_r:urandom_device_t:s0 
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0 
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0 
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0 
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
/var/run/ecblp0                                    named pipe         system_u:object_r:cupsd_var_run_t:s0 
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0 
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0 
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0 
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/usr/lib64 = /usr/lib
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/var = /var


Изменим тип контекста безопасности для каталога /etc/named: 

[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named


[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_zone_t:s0      121 Jun  2 13:06 .
drwxr-xr-x. 84 root root  system_u:object_r:etc_t:s0            8192 Jun  2 13:06 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_zone_t:s0   56 Jun  2 13:05 dynamic
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      784 Jun  2 13:05 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      610 Jun  2 13:05 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      609 Jun  2 13:05 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      657 Jun  2 13:05 named.newdns.lab

```





### Подключаемся к ВМ client и пробуем еще раз:
```

[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
> quit


[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9274
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: f5ff4cf7c41d632f010000006a1ed9776843bb46a356e1af (good)
;; QUESTION SECTION:
;www.ddns.lab.          IN  A

;; ANSWER SECTION:
www.ddns.lab.       60  IN  A   192.168.50.15

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Jun 02 13:24:07 UTC 2026
;; MSG SIZE  rcvd: 85
``` 






### Делаем ребут вмок и после ребута проверяем::
``` 

[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29615
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: f894ac8092fe6d67010000006a1eda329981b96828c1e4d3 (good)
;; QUESTION SECTION:
;www.ddns.lab.          IN  A

;; ANSWER SECTION:
www.ddns.lab.       60  IN  A   192.168.50.15

;; Query time: 10 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Jun 02 13:27:14 UTC 2026
;; MSG SIZE  rcvd: 85




Возвращаем как было

[root@ns01 ~]# restorecon -v -R /etc/named
Relabeled /etc/named from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.dns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.dns.lab.view1 from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic from unconfined_u:object_r:named_zone_t:s0 to unconfined_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab.view1 from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab.view1.jnl from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.newdns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.50.168.192.rev from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
[root@ns01 ~]# 


```
