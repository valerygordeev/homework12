# homework12
SELinux. Задание 2

Обеспечение работоспособности приложения при включенном SELinux

Вариант №1
1. Устанавливаем стенд из git clone https://github.com/mbfx/otus-linux-adm.git
2. Разворачиваем - ВМ посредством команды:
     vagrant up
3. Проверяем корректность установки ВМ:
     valery@lenovo:~/repo/otus-linux-adm/selinux_dns_problems$ vagrant status
     Current machine states:

     ns01                      running (virtualbox)
     client                    running (virtualbox)

     This environment represents multiple VMs. The VMs are all listed
     above with their current state. For more information about a specific
     VM, run `vagrant status NAME`.  
4. Подключамся к клиенту: 
     vagrant ssh client
5. Попробуем внести изменения в зону ddns:
     [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
     > server 192.168.50.10
     > zone ddns.lab
     > update add www.ddns.lab. 60 A 192.168.50.15
     > send
     update failed: SERVFAIL
     > quit
   Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.    
6. Для этого воспользуемся утилитой audit2why:
     [vagrant@client ~]$ sudo -i
     [root@client ~]# cat /var/log/audit/audit.log | audit2why
     Ничего не отобразилось, значит на клиенте ошибок нет.
7. Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:
     [vagrant@ns01 ~]$ sudo -i
     [root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
     type=AVC msg=audit(1707730775.459:1907): avc:  denied  { create } for  pid=5109 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	     Was caused by:
		     Missing type enforcement (TE) allow rule.

		     You can use audit2allow to generate a loadable module to allow this access.
		     
		     В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. 
8.  Проверим данную проблему в каталоге /etc/named:
      [root@ns01 ~]# ls -laZ /etc/named
      drw-rwx---. root named system_u:object_r:etc_t:s0       .
      drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
      drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
      -rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
      -rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
      -rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
      -rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
9.  Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:
      [root@ns01 ~]# semanage fcontext -l | grep named
      /etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
      /var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
      /etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
      /var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
      /var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
      /var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
      /var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
      /dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
      /var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0 
      /var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
      ....
      /usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0 
      /var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0 
      /var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0 
      /var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0 
      /var/named/chroot/lib64 = /usr/lib
      /var/named/chroot/usr/lib64 = /usr/lib
10.  Изменим тип контекста безопасности для каталога /etc/named:
       [root@ns01 ~]# chcon -R -t named_zone_t /etc/named
11.  Проверим изменение типа контекста:
       [root@ns01 ~]# ls -laZ /etc/named
       drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
       drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
       drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
       -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
       -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
       -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
       -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
12.  Перейдем на клиента и заново отправим изменения на dns:
       [root@client ~]# nsupdate -k /etc/named.zonetransfer.key
       > server 192.168.50.10
       > zone ddns.lab
       > update add www.ddns.lab. 60 A 192.168.50.15
       > send
       > quit
     Отправка дополнения была успешной
13.  Проверим работу домена dns:
       [root@client ~]# dig www.ddns.lab

       ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
       ;; global options: +cmd
       ;; Got answer:
       ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64068
       ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

       ;; OPT PSEUDOSECTION:
       ; EDNS: version: 0, flags:; udp: 4096
       ;; QUESTION SECTION:
       ;www.ddns.lab.			IN	A

       ;; ANSWER SECTION:
       www.ddns.lab.		60	IN	A	192.168.50.15

       ;; AUTHORITY SECTION:
       ddns.lab.		3600	IN	NS	ns01.dns.lab.

       ;; ADDITIONAL SECTION:
       ns01.dns.lab.		3600	IN	A	192.168.50.10

       ;; Query time: 1 msec
       ;; SERVER: 192.168.50.10#53(192.168.50.10)
       ;; WHEN: Mon Feb 12 10:37:41 UTC 2024
       ;; MSG SIZE  rcvd: 96
14.  Проконтролируем работу dns после перезагрузки клиента и сервера:
       [vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

       ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
       ; (1 server found)
       ;; global options: +cmd
       ;; Got answer:
       ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 383
       ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

       ;; OPT PSEUDOSECTION:
       ; EDNS: version: 0, flags:; udp: 4096
       ;; QUESTION SECTION:
       ;www.ddns.lab.			IN	A

       ;; ANSWER SECTION:
       www.ddns.lab.		60	IN	A	192.168.50.15

       ;; AUTHORITY SECTION:
       ddns.lab.		3600	IN	NS	ns01.dns.lab.

       ;; ADDITIONAL SECTION:
       ns01.dns.lab.		3600	IN	A	192.168.50.10

       ;; Query time: 3 msec
       ;; SERVER: 192.168.50.10#53(192.168.50.10)
       ;; WHEN: Mon Feb 12 10:40:31 UTC 2024
       ;; MSG SIZE  rcvd: 96
15. Восстановим типы dns на сервере к первоначальным значениям:
      [vagrant@ns01 ~]$ sudo -i
      [root@ns01 ~]# restorecon -v -R /etc/named
      restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
      restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
      restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0


Вариант №2
1. Повторям пункт 1-7.
2. Для решения проблемы создаем файла принудительного присвоения типов local.te:
     [root@ns01 ~]# cat /var/log/audit/audit.log | audit2allow -M local
     ******************** IMPORTANT ***********************
     To make this policy package active, execute:

     semodule -i local.pp
3. Компилируем политику: 
     [root@ns01 ~]# checkmodule -M -m -o local.mod local.te
     checkmodule:  loading policy configuration from local.te
     checkmodule:  policy configuration loaded
     checkmodule:  writing binary representation (version 19) to local.mod
4. Содержание модуля:
     [root@ns01 ~]# cat local.te

     module local 1.0;

     require {
	     type etc_t;
	     type named_t;
	     class file { create write };
     }

     #============= named_t ==============
     allow named_t etc_t:file write;

     #!!!! This avc is allowed in the current policy
     allow named_t etc_t:file create;
5. Собираем пакет:
     [root@ns01 ~]# semodule_package -o local.pp -m local.mod
6. Чтобы загрузить созданный пакет политики в ядро, устанавливаем скомпилированный модуль
     [root@ns01 ~]# semodule -i local.pp
7. После перезагрузки ВМ все работает:
   Клиент:
     [vagrant@client ~]$ sudo su
     [root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
     > server 192.168.50.10
     > zone ddns.lab
     > update add www.ddns.lab. 60 A 192.168.50.15
     > send
     > quit
   Сервер:
     [root@ns01 vagrant]# tail -n 50 /var/log/messages
     ...
     Feb 16 06:29:03 ns01 named[709]: client @0x7fea8409e160 192.168.50.15#56329/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
     Feb 16 06:29:03 ns01 named[709]: client @0x7fea8409e160 192.168.50.15#56329/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15
     Feb 16 06:29:29 ns01 su: (to root) vagrant on pts/0

Вариант №3

Вариант3
1. Редактируем playbook .../otus-linux-adm/selinux_dns_problems/provisioning/playbook.yml:
   Добавляем в задание:
   name: install packages # переведём синтаксис yum из deprecated  
   в packages:
   - python3-libselinux

   Добавляем перед заданием:
   name: ensure named is running and enabled:
      
  - name: Allow to modify context files in /etc/named/
    community.general.sefcontext:
      ftype: a
      serange: s0
      seuser: system_u
      setype: named_zone_t
      state: present
      target: '/etc/named(/.*)?'
      
  - name: Apply SELinux file context
    ansible.builtin.command: restorecon -irv /etc/named
2. Запускаем штатно vagrant
3. Обновляем записи dns сервера
   Клиент:
     [vagrant@client ~]$ sudo su
     [root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
     > server 192.168.50.10
     > zone ddns.lab
     > update add www.ddns.lab. 60 A 192.168.50.15
     > send
     > quit
4. Проверяем работу dns сервера
     [root@client vagrant]# dig www.ddns.lab

     ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
     ;; global options: +cmd
     ;; Got answer:
     ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11418
     ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

     ;; OPT PSEUDOSECTION:
     ; EDNS: version: 0, flags:; udp: 4096
     ;; QUESTION SECTION:
     ;www.ddns.lab.			IN	A

     ;; ANSWER SECTION:
     www.ddns.lab.		60	IN	A	192.168.50.15

     ;; AUTHORITY SECTION:
     ddns.lab.		3600	IN	NS	ns01.dns.lab.

     ;; ADDITIONAL SECTION:
     ns01.dns.lab.		3600	IN	A	192.168.50.10

     ;; Query time: 2 msec
     ;; SERVER: 192.168.50.10#53(192.168.50.10)есто
     ;; WHEN: Fri Feb 16 12:10:02 UTC 2024
     ;; MSG SIZE  rcvd: 96
5. Сервер:
   Проверим контекст директории /etc/named
     valery@lenovo:~/repo/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
     Last login: Fri Feb 16 11:33:24 2024 from 10.0.2.2
     [vagrant@ns01 ~]$ sudo su
     [root@ns01 vagrant]# ll -Z /etc/named
     drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
     -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
     -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
     -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
     -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
6. Просмострим логи обновления:
     Feb 16 11:37:22 localhost su: (to root) vagrant on pts/0
     Feb 16 11:39:59 localhost named[5243]: client @0x7f168c03c3e0 192.168.50.15#52461/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
     Feb 16 11:39:59 localhost named[5243]: client @0x7f168c03c3e0 192.168.50.15#52461/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15


Резюме:
Все варианты рабочие и имеют право на использование. Вариант отключения SELinux я даже не рассматриваю. 
Первый вариант, предоставленный в домашней работе, на мой взгляд самый прямолиненый. Второй вариант - более гибкий на мой взгляд. Вместо прямой замены, принудительно присваиваем типы: allow named_t etc_t:file write. Третий вариант, как я вижу, наиболее предпочтителен, т.к. предупреждает проблему и решая ее на этапе установки.







