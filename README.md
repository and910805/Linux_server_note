<!--
 * @Author: error: git config user.name && git config user.email & please set dead value or install git
 * @Date: 2022-10-27 13:17:30
 * @LastEditors: error: git config user.name && git config user.email & please set dead value or install git
 * @LastEditTime: 2022-11-23 07:22:57
 * @FilePath: \2上\note.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->

# 伺服器原理 筆記
作者標記 
李軒豪&&莊冠霖

[![redhat](https://img.shields.io/badge/Red_Hat--EE0000?style=plastic&logo=red-hat)](https://www.redhat.com/zh-tw/home)
[![apache](https://img.shields.io/badge/Apache--d22128?style=plastic&logo=apache)](https://httpd.apache.org/)
[![nginx](https://img.shields.io/badge/NGINX--009639?style=plastic&logo=nginx)](https://www.nginx.com/)
[![mariaDB](https://img.shields.io/badge/MariaDB--003545?style=plastic&logo=MariaDB)](https://mariadb.org/)
[![ansible](https://img.shields.io/badge/Ansible--ee0000?style=plastic&logo=ansible)](https://www.ansible.com/)
[![HTML5](https://img.shields.io/badge/HTML5--e34f26?style=plastic&logo=HTML5)](https://zh.wikipedia.org/zh-tw/HTML5)
[![Dovecot](https://img.shields.io/badge/Dovecot--0078d7?style=plastic&logo=Dovecot)](https://www.dovecot.org/)
[![PHP](https://img.shields.io/badge/PHP--777bb4?style=plastic&logo=PHP)](https://zh.m.wikipedia.org/zh-hant/PHP)
[![MySQL](https://img.shields.io/badge/MySQL--4479a1?style=plastic&logo=MySQL)](https://www.mysql.com/)
[![PHPMyAdmin](https://img.shields.io/badge/PHPMyAdmin--f37623?style=plastic&logo=PHPMyAdmin)](https://www.phpmyadmin.net/)
[![bash](https://img.shields.io/badge/Bash--4eaa25?style=plastic&logo=GNU-Bash)](https://www.gnu.org/software/bash/)

## 要使用的機器

- a (dns server)(172.255.250.10)
- b (mail server)
- c (web server)
- d (load balance server)
- bastion
- worksataion

## dns setup notes

### universal

- change resolv.conf to new dns server

```bash
vim /etc/resolv.conf
```

### file: /etc/resolv.conf

```apacheconf
ip = 172.250.25.10 
#dns server
```

### workstation

```bash
$ssh root@servera # login servera
```

### servera

```bash
$yum install bind-chroot.x86_64 # install bind
$systemctl start named # start bind
$systemctl enable named # enable bind
$systemctl status named # check bind status
$firewall-cmd --list-all # check firewall status
$firewall-cmd --permanent --add-service=dns # add dns service to firewall
$firewall-cmd --reload # reload firewall
$firewall-cmd --list-all # check firewall status ps: dns should be added
$systemctl restart named-chroot # restart bind
$vim /etc/named.conf # edit named.conf
```

### file: /etc/named.conf

```bash
options : listen-on port 53 { 172.25.250.10 }; #ip of servera
options : allow-query{any;}; # allow any ip to query#考試是會規定只有一個server可以使用
#讓slave那台server對外，達到安全
```

使用host指令查看dns server是否正常

```bash

$host www.google.com

```

### servera

編輯 `named.conf` 檔案，增加新的 zone

```bash
$systemctl restart named-chroot #restart bind
$vim /etc/named.zones # or edit named.rfc1912.zones
```

### file: /etc/named.zones

```apacheconf
zone "example.com" IN {
    type master;
    file "example.com.zone";
    };
```

>上面的 `example.com` 是我們要設定的網域名稱，`example.com.zone` 是我們要設定的網域名稱的 zone 檔案。</br>
>名子可以自己取，但是要注意的是，檔案的副檔名要是 `.zone`。

### servera

```bash
cd /var/named
cp named.empty example.com.zone #這裡用複製的方式可以避免打錯字，然後後面的檔案名稱要跟上面的 zone 檔案名稱一樣
chgrp named example.com.zone #修改檔案的群組
vim example.com.zone #編輯 zone 檔案
```

### file: /var/named/example.com.zone

```bash
$TTL 3600 #TTL是指定這個檔案的資料的有效時間，也就是說，如果你的網域名稱的 IP 有變動，那麼這個檔案的資料就會在 3600 秒後失效，會需要重新請求 DNS server。
$ORIGIN example.com. #若以下網址沒有以 . 結束，將會自動接上 $ORIGIN。
@ IN SOA dns.example.com. hibana2077.gmail.com (
    2022103101 ; serial #這個是指定這個檔案的版本，每次修改完這個檔案後，都要將這個數字加一，這樣才能讓 DNS server 知道這個檔案有更新。
    3H ; refresh #slave 每隔這段時間就會確認 master’s Serial。
    15 ; retry #slave 無法連結 master 時，要隔多久重試。
    1W ; expire #slave 無法連結 master 時，要等多久才會判斷 master 已經死亡。
    1D ; minimum #代表這份 noze file 中所有 record 的內定 TTL 值，也就是這份資料的暫存時間不會超過這個時間。
    )
        NS @
        A 172.25.250.10
www     IN A    172.25.250.11
mail    IN A    172.25.250.11
mail    IN MX   10 mail.example.com.
```


### servera

```bash
$systemctl restart named-chroot #restart bind
$systemctl status named-chroot #check bind status
```

### workstation

```bash
$host www.example.com #check dns server
```

如果成功的話，會看到類似這樣的結果，這樣就代表 dns server 已經設定完成了。

```bash
www.example.com has address 
```

***

## DNS 正向解析

### servera

```bash
$vim /var/named/example.com.zone #編輯 zone 檔案
```

### file: /var/named/example.com.zone

```bash
$TTL 3600 #TTL是指定這個檔案的資料的有效時間，也就是說，如果你的網域名稱的 IP 有變動，那麼這個檔案的資料就會在 3600 秒後失效，會需要重新請求 DNS server。
$ORIGIN example.com. #若以下網址沒有以 . 結束，將會自動接上 $ORIGIN。
@ IN SOA dns.example.com. hibana2077.gmail.com (
    2022103101 ; serial #這個是指定這個檔案的版本，每次修改完這個檔案後，都要將這個數字加一，這樣才能讓 DNS server 知道這個檔案有更新。
    3H ; refresh #slave 每隔這段時間就會確認 master’s Serial。
    15 ; retry #slave 無法連結 master 時，要隔多久重試。
    1W ; expire #slave 無法連結 master 時，要等多久才會判斷 master 已經死亡。
    1D ; minimum #代表這份 noze file 中所有 record 的內定 TTL 值，也就是這份資料的暫存時間不會超過這個時間。
    )
        NS @
        A 172.25.250.10
www IN  A 172.25.250.11
db  IN  A 172.25.250.12
mail IN A 172.25.250.13
```

### servera

```bash
$systemctl restart named-chroot #restart bind
$systemctl status named-chroot #check bind status
```

### workstation

```bash
$host www.example.com #check dns server
$host db.example.com #check dns server
```

***

## DNS 反向解析

### servera

現在要去 `/etc/named.conf` 設定反向解析的設定檔。

```bash
$vim /etc/named.conf
```

### file: /etc/named.conf

```apacheconf
zone "250.25.172.in-addr.arpa" IN {
    #注意 這裡的IP要反過來寫
    type master;
    file "example.rr.file";
};
```

### servera

```bash
$cp /var/named/named.empty /var/named/example.rr.file #複製一個空的檔案
$chgrp named /var/named/example.rr.file #修改檔案的群組
$vim /var/named/example.rr.file #編輯反查檔案
```

### file: /var/named/example.rr.file

```bash
$TTL 3H
$ORIGIN 250.25.172.in-addr.arpa. #注意 這裡的IP要反過來寫
@       IN SOA example.com. hibana2077.gmail.com. (
        2021103101 ; serial
        3H ; refresh
        15 ; retry
        1W ; expire
        1D ; minimum
        )
        NS example.com.
10      IN PTR dns.example.com.
11      IN PTR www.example.com.
12      IN PTR db.example.com.
13      IN PTR mail.example.com.
```

### servera

```bash
$systemctl restart named-chroot #restart bind
$systemctl status named-chroot #check bind status
```

### workstation

測試反向解析

```bash
$host db.example.com #用正向解析查詢IP
>> db.example.com has address 172.25.250.12
$host 172.25.250.12 #用反向解析查詢網域名稱
>> 12.250.25.172.in-addr.arpa domain name pointer db.example.com.#如果換成13 就會顯示 mail.example.com
```

**小記**

> 考試的時候好像會需要再用server b 開一個dns server，然後把server c 的dns server改成server b的ip。

***
## Slave主僕伺服器
### serverb
```bash
$yum install bind-chroot.x86_64 # install bind
$systemctl start named # start bind
$systemctl enable named # enable bind
$systemctl status named # check bind status
$firewall-cmd --list-all # check firewall status
$firewall-cmd --permanent --add-service=dns # add dns service to firewall
$firewall-cmd --reload # reload firewall
$firewall-cmd --list-all # check firewall status ps: dns should be added
$systemctl restart named-chroot # restart bind
$vim /etc/named.conf # edit named.conf
```
### file: /etc/named.conf #serverb

```bash
options : listen-on port 53 { 172.25.250.11; }; #ip of servera
options : allow-query{any;}; # allow any ip to query
zone "example.com" IN {#正查
    type slave;
    file "example.com.zone";
    masters{172.25.250.10; };
    };
zone "250.25.172.in-addr.arpa" IN {#反查
    type slave;
    file "example.rr.zone";
    masters{172.25.250.10; };
    };
```
### file: /etc/named.conf #servera
```bash
zone "250.25.172.in-addr.arpa" IN {
    #注意 這裡的IP要反過來寫
    type master;
    file "example.rr.file";
    allow-transfer{172.25.250.11}#serverb的ip
};
zone "example.com" IN {
    type master;
    file "example.com.zone";
    allow-transfer{172.25.250.11}#serverb的ip
    };
```
### Serverb
```bash
$systemctl restart named-chroot
```
### workstation
```bash
$vim /etc/resolv.conf
改成b的ip
$host 172.25.250.10
```
## Unbound

`bind` 跟 `unbound` 都是 dns server，但是 `bind` 會有一些問題，所以我們會用 `unbound` 來取代 `bind。`

### workstation

先把 workstation 的 dns server 改成 serverc 的 ip

```bash
$vim /etc/resolv.conf #編輯 resolv.conf
```

### file: /etc/resolv.conf

```apacheconf
nameserver 172.25.250.12
```

### workstation

檢查是否有改成功

```bash
$hose www.example.com
>> www.example.com has address ip
```

### server c

安裝 unbound

```bash
$yum install unbound #安裝 unbound
$systemctl start unbound #啟動 unbound
$systemctl enable unbound #設定 unbound 開機自動啟動
$systemctl status unbound #檢查 unbound 狀態
$firewall-cmd --permanent --add-service=dns #開啟防火牆的 dns 服務
$firewall-cmd --reload #重新載入防火牆
$unbound-control-setup #設定 unbound
```

更改 unbound 的設定檔

```bash
$vim /etc/unbound/unbound.conf
```

### file: /etc/unbound/unbound.conf

```apacheconf
interface: 172.25.250.12 #設定 unbound 的 ip#49
access-control: 172.25.250.0/24 allow #允許的IP#255
forward-zone:#874
    name: "example.com"
    forward-addr:172.25.250.10 
    #(servera的ip)
```

### server c

```bash
$unbound-checkconf #檢查設定檔
$systemctl restart unbound #restart unbound
$systemctl status unbound #check unbound status
$unbound-control dump_cache #查看 unbound 的 cache
$unbound-control dump_cache | grep example.com #查看 example.com 的 cache
$unbound-control flush_cache #清除 unbound 的 cache
$unbound-control dump_cache > test.dnsdump #清除 unbound 的 cache
#導出的位置可以換成其他地方
$unbound-control load_cache < test.dnsdump #導入 unbound 的 cache
```

> Tips: 如果覺得沒有清除成功，可以用 `systemctl restart unbound` 來重啟 unbound。

### workstation

用 `dig` 測試 unbound 是否有正常運作

```bash
$dig www.example.com
```

成功的話會顯示

```bash
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7_9 <<>> www.example.com

```

這樣開頭的字串。

#### 正向查詢

```bash
$dig www.example.com
```

***

## Email Server

使用 `postfix` 跟 `dovecot` 來建立 email server。

`Postfix` 是 `SMTP` 伺服器，負責處理郵件的傳送。

`Dovecot` 可使用 `POP3 / IMAP` 服務 ， 負責接收郵件跟郵件的身份驗證。

`OpenDkim` (DKIM：DomainKeys Identified Mail) 是使用簽章做認證的電子郵件，目的是改善垃圾、釣魚郵件問題。

伺服器：
- servera
- serverb

### serverb

安裝 `postfix`

```bash
$yum install postfix #安裝 postfix
$systemctl start postfix #啟動 postfix
$systemctl enable postfix #設定 postfix 開機自動啟動
$systemctl status postfix #檢查 postfix 狀態
```

### servera

開一個網址為 `mail.example.com` 的網站，並且把 `mail.example.com` 的 A 記錄指向 `serverb` 的 ip。

```bash
$vim /var/named/example.com.zone #編輯 zone 檔
```

### file: /var/named/example.com.zone

```bash
$TTL 3H
$ORIGIN example.com.
@       IN SOA @ example.com. (
                        2022111301 ; serial
                        3H ; refresh
                        1H ; retry
                        1W ; expire
                        1D ) ; minimum
        NS      @
        A       172.25.250.10

www     IN A    172.25.250.11
mail    IN A    172.25.250.11
mail    IN MX   10 mail.example.com.
```

### servera

```bash
$systemctl restart named #restart named
```

### serverb

```bash
$host mail.example.com #檢查 dns 是否有設定成功
$vim /etc/postfix/main.cf #編輯 postfix 的設定檔
```

### file: /etc/postfix/main.cf

```apacheconf
myhostname = mail.example.com #設定主機名稱#95
mydomain = example.com #設定 domain#102
myorigin = $mydomain #設定 origin#117
inet_interfaces = $myhostname #設定網路介面#133
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain #設定目的地
mynetworks = 172.25.250.0/24,172.0.0.0/8 #設定允許的網路 => 同一個網段#286
```

![image](https://media.discordapp.net/attachments/964711672411467777/1044574611871895622/image.png?width=1158&height=651)

### serverb

```bash
$systemctl restart postfix #restart postfix
$systemctl status postfix #檢查 postfix 狀態
```

![image](https://media.discordapp.net/attachments/964711672411467777/1044574890826674206/image.png)

### serverb

```bash
$firewall-cmd --permanent --add-service=smtp #開啟 smtp 服務
$firewall-cmd --reload #重新載入防火牆
#測試
$adduser test #新增一個使用者
$passwd test #設定密碼
```

### workstation

```bash
$yum install postfix #安裝 postfix
$systemctl start postfix #啟動 postfix
$systemctl enable postfix #設定 postfix 開機自動啟動
$yum install mailx #安裝 mailx
```

![img](https://media.discordapp.net/attachments/716638961946066975/1044577495032283176/image.png)

### workstation

```bash
#格式: mail -s "主旨" 收件者
$mail -s "Hello" test@mail.example.com
Hello
This is a test mail.
.
EOT #<= 輸入.後按下 enter 會出現這個
```

### serverb

![arc](https://media.discordapp.net/attachments/716638961946066975/1044578585471627264/image.png)

```bash
$cat /var/spool/mail/test #檢查是否有收到郵件
```

![res](https://media.discordapp.net/attachments/716638961946066975/1044579144740110396/image.png)

### serverb

```bash
$yum install mailx #安裝 mailx
$su - test #切換到 test 使用者
$mail #打開 mail，然後會看到收到的郵件，按下 q 離開，輸入編號就可以看到該郵件的內容
```

## Postfix 更改變數

```bash
$postfix #查看 postfix 的所有變數
$postconf inet_interfaces #查看 inet_interfaces 的值
$postconf -e "myorigin = example.com" #設定 myorigin 的值
```

## SPF

產生器連結[link](https://mxtoolbox.com/SPFRecordGenerator.aspx)

![img](https://media.discordapp.net/attachments/740572974456766486/1044585178628108348/image.png)

![img](https://media.discordapp.net/attachments/716638961946066975/1044585350607159367/image.png)

### workstation

```bash
$ssh servera #連線到 servera(DNS)
```

### servera

```bash
$su - 
$vim /var/named/example.com.zone #編輯 zone 檔
```

### file: /var/named/example.com.zone

```apacheconf
$TTL 3H
$ORIGIN example.com.
@       IN SOA @ example.com. (
                        2022111301 ; serial
                        3H ; refresh
                        1H ; retry
                        1W ; expire
                        1D ) ; minimum
        NS      @
        A       172.25.250.10

www     IN A    172.25.250.11
mail    IN A    172.25.250.11
mail    IN MX   10 mail.example.com.
        IN TXT  "v=spf1 a mx ip4:172.25.250.0/24 ~all"
```

### servera

```bash
$systemctl restart named-chroot #restart named-chroot
```

## DKIM

產生器連結[link](https://dkimcore.org/tools/)

![img](https://media.discordapp.net/attachments/716638961946066975/1044588549963522088/image.png?width=659&height=652)

### workstation

```bash
$ssh serverc #連線到 serverc
```

### serverc

```bash
$vim /etc/resolv.conf #編輯 resolv.conf => 將 nameserver 改成 servera 的 IP
$yum install postfix
$systemctl start postfix
$systemctl enable postfix
$systemctl status postfix
$vim /etc/postfix/main.cf #編輯 main.cf
```

### file: /etc/postfix/main.cf

```apacheconf
inet_interfaces = loopback-only #設定為 loopback-only#136
myorigin = serverc.lab.example.com #設定 myorigin#118
mynetworks = 127.0.0.0/8 #設定 mynetworks#286
local_transport = error:local delivery disabled #設定 local_transport
mydestination = #184
relayhost = [mail.example.com] #設定 relayhost#340
```

### serverc

```bash
$systemctl restart postfix #restart postfix
$systemctl status postfix #檢查 postfix 狀態
```

可以用

```bash
$postfix inet_interfaces myorigin relayhost mydestination local_transport mynetworks
```

來查看設定的值，應該要和下圖一樣 P.S relayhost 要跟自己的mail server的名稱一樣

![check](https://media.discordapp.net/attachments/716638961946066975/1044600379410743356/image.png)

## dovecot

- workstation
- serverb
- serverc

### serverb

```bash
$systemctl start postfix #啟動 postfix
$systemctl status postfix #檢查 postfix 狀態
$yum install dovecot #安裝 dovecot
$systemctl start dovecot #啟動 dovecot
$systemctl status dovecot #檢查 dovecot 狀態
$systemctl enable dovecot #設定 dovecot 開機自動啟動
$cd /etc/dovecot/
$vim dovecot.conf #編輯 dovecot.conf
```

### file: /etc/dovecot/dovecot.conf

```apacheconf
#設定 listen => mail server 的 IP
listen = 172.25.250.11#30
login_trusted_networks = 172.25.250.0/24#48
protocols = imap pop3#24
```

### serverb

```bash
$vim /etc/dovecot/conf.d/10-mail.conf #編輯 10-mail.conf
```

### file: /etc/dovecot/conf.d/10-mail.conf

```apacheconf
mail_location = mbox:~/mail:INBOX=/var/spool/mail/%u#25
```

### serverb

```bash
$cat /etc/dovecot/dovecot.conf | grep -v "#" | grep protocols #檢查 protocols
$cat /etc/dovecot/doveconf.conf | grep -v "#" | grep listen #檢查 listen
$cat /etc/dovecot/dovecot.conf | grep -v "#" | grep login_trusted_networks #檢查 login_trusted_networks
$cat /etc/dovecot/conf.d/10-mail.conf | grep -v "#" | grep mail_location #檢查 mail_location
$systemctl restart dovecot #restart dovecot
$systemctl status dovecot #檢查 dovecot 狀態
```

![img](https://media.discordapp.net/attachments/716638961946066975/1044609266184626206/image.png)

```bash
$cd /var/spool/mail/
$ls -l
$chmod g-rw *
$ls -l
$firewall-cmd --permanent --add-service=pop3 #設定防火牆
$firewall-cmd --permanent --add-service=imap #設定防火牆
$firewall-cmd --reload #重新載入防火牆
$systemctl restart dovecot #restart dovecot
$systemctl status dovecot #檢查 dovecot 狀態
```

### serverc

```bash
$yum install mutt #安裝 mutt
$mutt -f imap://test@mail.example.com #測試 mutt
```

### mutt

```bash
/root/Mail does not exist. Create it? (y/n) y #輸入 y
SSL Certificate check : a
Password for test@mail.example.com: #輸入密碼
```

![img](https://media.discordapp.net/attachments/716638961946066975/1044612335559528488/image.png)

### serverc

```bash
$mutt -f pop://test@mail.example.com
```

![img](https://media.discordapp.net/attachments/716638961946066975/1044612335559528488/image.png)

### pop3

```bash
Password for test@mail.example.com
```


以上是mail server的部分，接下來是web server的部分。

***

## web server

- workstation
- servera (dns server)
- serverc (web server)

## httpd

### serverc

```bash
$yum install httpd #安裝 httpd
$systemctl start httpd #啟動 httpd
$systemctl status httpd #檢查 httpd 狀態
$systemctl enable httpd #設定 httpd 開機自動啟動
```

### servera

```bash
$vim /var/named/example.com.zone #編輯 example.com.zone
```

### file: /var/named/example.com.zone

```apacheconf
$TTL 3H
$ORIGIN example.com.
@       IN SOA @ example.com. (
                        2022112201 ; serial
                        3H ; refresh
                        1H ; retry
                        1W ; expire
                        1D ) ; minimum
        NS      @
        A       172.25.250.10

mail    IN A    172.25.250.11
mail    IN MX   10 mail.example.com.
        IN TXT  "v=spf1 a mx ip4:172.25.250.0/24 ~all"

www     IN A    172.25.250.12
#172.25.250.12 是 web server 的 IP
```

### servera

```bash
$systemctl restart named #restart named
$systemctl status named #檢查 named 狀態
```

### serverc

```bash!
$host www.guanlin.tw #需指到12
$firewall-cmd --permanent --add-service=http #設定防火牆
$firewall-cmd --permanent --add-service=https #設定防火牆
$firewall-cmd --reload #重新載入防火牆
$cd /var/www/html/
$vim index.html #編輯 index.html
    Hello!
$vim /etc/httpd/conf/httpd.conf
    可以在這裡改更目錄
    Listen 172.25.250.12:80#45
    ServerAdmin guanlin@mail.guanlin.tw#89
```
### servera
```bash
$vim /etc/named.conf 
$copy =>zone"family.tw"
$cd /var/named
$cp guanlin.tw.zone family.tw.zone
$chgrp named guanlin2.tw.zone
$vim family.tw.zone 記得去改成family
$systemctl restart named-chroot
```

### serverc

```bash
$host www.family.tw #要找的到and guanlin
$cd /var/www/html
$mkdir guanlin#資料夾名稱
$mkdir family
$cd guanlin
$vim index.html
    guanlin
$cd ../family
$vim index.html
    family
$cd /etc/httpd/conf.d
$vim guanlin.conf
```
### /etc/http/conf.d/guanlin.conf 
```
<Directory /var/www/html/guanlin>
    AllowOverride None
    Require all granted
</Directory>
<VirtualHost *:80>
    DocumentRoot /var/www/html/guanlin
    ServerAdmin guanlin@mail.guanlin.tw
    ServerName www.guanlin.tw
    Errorlog "logs/guanlin_error.log"
    CustomLog "logs/guanlin_access.log"combined
</VirtualHost>
```
### serverc
```bash
$cp guanlin.conf family.conf
$vim  family.conf
    修改成family
$systemctl restart httpd

```
### workstation

```bash
$curl http://www.guanlin.com #測試 httpd
```

或是開firefox瀏覽器，輸入網址 http://www.guanlin.com

### serverc

```bash
vim /etc/httpd/conf/httpd.conf #編輯 httpd.conf
```

### file: /etc/httpd/conf/httpd.conf

```apacheconf
ServerAdmin test@mail.example.com
```
### serverc

```bash
$systemctl restart httpd #restart httpd
$systemctl status httpd #檢查 httpd 狀態
```


如果在 `/var/www/html/` 目錄以外的地方建立html檔案，需要修改 `/etc/httpd/conf/httpd.conf` 的路徑設定，跟SELinux的設定。


## 動態網頁


serverc
```bash
$yum install php
$yum install php*
$vim /etc/httpd/conf/httpd.conf 
    index.php index.html #167
$systemctl restart httpd #restart httpd
$cd /var/www/html/guanlin
$vim index.php
```
```php
<?php
    phpinfo()
?>
```
## nginx 裝在serverc要記得關httpd
```bash
systemctl stop httpd
systemctl disable httpd
yum install nginx
systemctl restart nginx
systemctl enable nginx
cd /usr/share/nginx/html#放網站的位置
```
兩個資料夾根目錄設定 
vim /etc/nginx/nginx.conf
```nginx
server{
    listen 80;
    server_name www.guanlin.tw;
    root /var/www/html/guanlin;
}
server{
    listen 80;
    server_name www.family.tw;
    root /var/www/html/family;
}
```
```bash
systemctl restart nginx
```
## https憑證
### workstation
```bash
$mkdir gl
```
### 外面終端機
```bash!
將ptt playbook的內容複製到外面終端機然後縮排改名字
scp playbook.yml root@workstation:~/gl/playbook.yml
```
### workstation
```bash
$cd gl 
$vim ansible.cfg
```
```bash
[defaults]
inventory=./inventory
remote_user=root
ask_pass=false
[privilege_escalation]
become=true
become_method=sudo
become_user=root
become_ask_pass=true
```
```bash
$vim inventory
```
```bash
[gl]
www.gl.tw
```
```bash
$ansible-playbook playbook.yml
```
### serverc
```bash
$yum install mod_ssl
$cd /etc/httpd/conf.d
$vim gl.conf
```
```bash
<VirtualHost *:80>
    ServerName www.gl.tw
    Redirect "/" https://www.gl.tw
</VirtualHost>
80 -> 443
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/gl.crt
SSLCertificateKeyFile /etc/pki/tls/private/gl.key
```
```bash
$systemctl restart httpd
查看瀏覽器
```

## varnish  反向代理 
### serverc
```bash
$yum install varnish
$systemctl restart varnish
$cd /etc/httpd/conf
$vim httpd.conf
    Listen 8888#45
$cd ../conf.d
$mv guanlin.conf guanlin.bak
$mv family.conf family.bak
$cd /var/www/html
$vim index.html
    hello!
$semanage port -a -t http_port_t -p tcp 8888
$systemctl stop nginx
$systemctl restart httpd
$firewall-cmd --add-port=8888/tcp
#上網http://www.guanlin.tw:8888
$cd /etc/systemd/system/
$mkdir varnish.service.d
$cd varnish.service.d
$vim httpport.conf
    [Service]
    ExecStart=
    ExecStart=/usr/sbin/varnishd -a :80 -f /etc/varnish/default.vcl -s malloc,256m
$systemctl daemon-reload#重載設定
$systemctl restart varnish
$firewall-cmd --permanent --add-service=http
$firewall-cmd --reload
$setsebool -P varnishd_connect_any on#這個bool打開
$cd /etc/varnish
$vim default.vcl
    8888 #18
$systemctl restart varnish#可以www.guanlin.tw查查看
$firewall-cmd --remove-port=8888/tcp
```
## 負載平衡haproxy

### serverb
```bash
$yum isntall haproxy
$cd /etc/haproxy
$vim haproxy.cfg
    bind *:80#68
    default_backend 改guanlin#73
    
    backend guanlin
        balance roundrobin
        server www1 172.25.250.12
        server www2 172.25.250.13
$systemctl restart haproxy
$firewall-cmd --permananent --add-service=http
$firewall-cmd --reload
```
### servera
```bash
cd /var/named
vim guanlin
改www 11
```
### serverc
```bash
$firewall-cmd --permanent --add-source=172.25.250.11 --zone=internal
$firewall-cmd --permanent --add-service=http --zone=internal
$firewall-cmd --reload
```
### serverd
```bash
$vim /etc/resolv.conf
    改10
$yum install httpd
$systemctl restart httpd
$cd /var/www/html
$vim index.html
        123456
$firewall-cmd --permanent --add-source=172.25.250.11 --zone=internal
$firewall-cmd --permanent --add-service=http --zone=internal
$firewall-cmd --reload    
```
### serverb
```bash
$curl http://172.25.250.13/#可以去做測試
```
## DB 
### serverd
```bash
$yum install mariadb
$yum install mariadb-server
$systemctl restart mariadb 
$mysql_secure_installation
enter->redhat y,y,y,y,y
$cd /etc/my.cnf.d
$vim mariadb-server.cnf
    copy 37->21 172.25.250.13
$systemctl restart mariadb
$firewall-cmd --permanent --add-service=mysql
$firewall-cmd --reload
$mysql -p #連線輸入密碼
$show databases;
$use mysql;#進入mysql
$show tables;
$describe user; 描述user表
$select Host,Passwoed,User from user;
$create database MY_DB;
$use MY_DB;
$create table product(
    id int(11) not null primary key auto_increment,
    name varchar(100) not null,
    price double not null,
    stock int(11) not null,
    id_category int(11) not null,
    id_manufacturer int(11) not null);
$show tables;
$describe product;
```
### insert
```bash!
$insert into product(name,price,stock,id_category,id_manufacturer)values('a',46.15,20,2,4);
```
### select
```bash!
$select * from product;#查看所有
```
### delete
```bash!
$delete from product where id=2;#刪掉2
```
### update
```bash!
$update product set id=2 where id=4;#改第四個的id為2
$update product set stock=0 where id=4;#改第四個的stock
```
### 改權限
```bash
$use mysql;
$select Host,User,Password from user;
$create user guanlin@localhost identified by 'redhat';#在本地登入的guanlin使用者
$create user guanlin@172.25.250.12 identified by 'redhat';#在172.25.250.12登入的guanlin使用者
$grant all privileges on *.* to guanlin@localhost;#給所有資料庫
$grant all privileges on MY_DB.* to guanlin@localhost;#給他MY_DB裡面的所有資料表
$grant all privileges on MY_DB.product to guanlin@localhost;#給他MY_DB裡面的product這個資料表
$grant select on *.* to guanlin@localhost;#給查詢功能而已
$flush privileges;
$show grants for guanlin@loalhost;#查看本地端權限
$show grants for guanlin@172.25.250.12;
$exit
```
### serverc
```bash=
$yum install mariadb
$mysql -u guanlin -p -h 172.25.250.13#-u 使用者 -p密碼-h 遠端
$show database;
$use MY_DB;
$show tables;
$select * from product;
```
### DB備份(serverd)
```bash!
$mysqldump -u guanlin -p MY_DB
$mysqldump -u guanlin -p MY_DB>MY_DB_20203.backup#備份到..資料夾
$mysqldump -p --all-databases>MY_DB_20203.backup#全部資料庫備份
```
### DB還原
```bash!
$mysql -p
$drop database MY_DB;
$create database MY_DB;
$mysql -p MY_DB < MY_DB_20203.backup
```


