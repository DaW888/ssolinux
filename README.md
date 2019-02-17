# Sprawdzian Linux - DHCP, DNS, FTP, SSH, WWW

> nano /etc/nanorc : `set linenumbers`

## DHCP (konfiguracja kart sieciowych)

> **instalacja**: apt install isc-dhcp-server
---
>karty sieciowe ustawione na: 1.NAT, 2.INTERNAL

### wazne pliki

- /etc/network/interfaces
- /etc/default/isc-dhcp-server
- /etc/dhcp/dhcpd.conf

### ważne komendy

- `ip a`
- `systemctl restart networking`
- `systemctl status/restart isc-dhcp-server`
- `cp dhcpd.conf dhcpd.conf.old` - dla zabezpieczenia

### pliki

#### /etc/network/interfaces

```bash
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

#Primary - NAT
allow-hotplug enp0s3
auto enp0s3
iface enp0s3 inet dhcp

#Internal
auto enp0s8
iface enp0s8 inet static
address 192.168.0.1
netmask 255.255.255.0
```

- auto eth0 – podnosi dany interfejs
- iface eth0 inet static – określa, czy adres IP ma być przypisywany statycznie czy dynamicznie (tutaj akurat ma być
statycznie)
- address – adres IP danego interfejsu sieciowego
- netmask – maska podsieci
- gateway – bramka
- network – określa, do której sieci ma przynależeć dana karta sieciowa. Ta opcja będzie nam bardzo potrzebna przy
konfiguracji DHCP.

#### /etc/default/isc-dhcp-server

```bash
INTERFACESv4="enps8" # czyli Internal
```

#### /etc/dhcp/dhcpd.conf

```bash
subnet 192.168.0.0 netmask 255.255.255.0{
    range 192.168.0.6 192.168.0.35; # podajemy zakres przydziału dhcp
    option routers 192.168.0.1;
    option domain-name "test.pl";
    option domain-name-servers 192.168.0.1 8.8.8.8; # pierwszy to adres karty Internal a drugi to adres googla
    default-lease-time 43200;
    max-lease-time 86400;
}
host user2{
    hardware ethernet 08:00:27:AF:ED:7E;
    fixed-address 192.168.0.6;
    #przydzielenie stałego adresu ip po MACu
}
```

---

## DNS

> **instalacja**: apt install bind9 dnsutils bind9-doc

### ważne pliki

- /etc/bind/db.local
- /etc/bind/named.conf.defalt-zones
- /etc/resolv.conf

### ważne komendy

- `cp db.local [nazwa].local`
- `systemctl restart bind9`
- `nslookup [gandalf.ring.hf]/[192.168.0.1]`

### pliki

#### /etc/bind/db.ring

```bash
$TTL    300
@       IN      SOA     gandalf.ring.hf.    root.ring.hf. (
                            2               ; Rerial
                        21600               ; Refresh
                        10800               ; Retry
                        172800              ; Expire
                        604800 )            ; Negative Cache TTL
;
@       IN      NS      gandalf.ring.hf.
@       IN      A       192.168.0.1

gandalf.ring.hf.        IN      A       192.168.0.1
ns1                     IN      CNAME   gandalf.ring.hf.

frodo.ring.hf.          IN      A       192.168.0.2
@                       IN      MX      20      frodo.ring.hf.
mail                    IN      CNAME   frodo.ring.hf

gimli.ring.hf.          IN      A       192.168.0.5
ftp                     IN      CNAME   gimli.ring.hf.

# to Po WWW

a       IN      CNAME       ring.hf.
b       IN      CNAME       ring.hf.
c       IN      CNAME       ring.hf.
d       IN      CNAME       ring.hf.
```

#### /etc/resolv.conf

```bash
nameserver 192.168.0.1 # adres gandalfa - karty Internal
```

#### /etc/bind/named.conf.default-zones

```bash
zone "ring.hf"{
    type master;
    file "/etc/bind/db.ring";
    allow-transfer {192.168.0.6;}; # ip naszego klienta
};

# to wpisujemy po edycji plików na kliencie
zone "0.168.192.in-addr.arpa"{
    type slave;
    file "/etc/bind/db.arpa";
    masters {192.168.0.6;};
};
```

### KLIENT

> Karta Sieciowa : 1.NAT 2.Internal

#### pliki

/etc/network/interfaces

```bash
auto enp0s3
allow-hotplug enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
address 192.168.0.29
netmask 255.255.255.0
```

**tworzymy plik db.arpa w `/etc/bind/`**

```bash
$TTL    604800
@       IN      SOA     aragon.ring.hf.    root.ring.hf. (
                            2               ; Rerial
                       604800               ; Refresh
                        86400               ; Retry
                      2419200              ; Expire
                        604800 )            ; Negative Cache TTL
;
@       IN      NS      gandalf.ring.hf.
@       IN      NS      aragon.ring.hf.

1.0.168.192.in-addr.arpa.       IN      PTR     gandalf.ring.hf
2.0.168.192.in-addr.arpa.       IN      PTR     frodo.ring.hf
3.0.168.192.in-addr.arpa.       IN      PTR     sam.ring.hf
4.0.168.192.in-addr.arpa.       IN      PTR     legolas.ring.hf
5.0.168.192.in-addr.arpa.       IN      PTR     gimli.ring.hf
6.0.168.192.in-addr.arpa.       IN      PTR     aragon.ring.hf
```

**edytujemy plik `named.conf.default-zones` w `/etc/bind/`**

```bash
zone 0.168.192.in-addr.arpa {
    type master;
    file "/etc/bind/db.arpa";
    allow-transfer {192.168.0.1;};
};

zone "ring.hf" {
    type slave;
    file "/etc/bind/db.ring";
    masters {192.168.0.1;};
};
```

---

## FTP

> **instalacja**: apt install vsftpd
---
> **instalacja klient**: apt install ftp

### ważne pliki

- /etc/vsftpd.conf

### ważne komendy

- `cp vsftpd.conf vsftpd.conf.old`
- `man 5 vsftpd.conf`
- `systemctl restart vsftpd`
- `open 192.168.0.1 222` - open [ip port]
- `man ftp`
- `ftp localhost`

### komendy zarzadzania plikami w ftp

- `bye` - konczy proces ftp
- `cd zdalny_katalog`
- `close` - zamyka polaczenie, nie zabija procesu
- `delate zdalny_plik`
- `dir [zdalny_kat] [plik_lokalny]`
- `put [lokalny_plik] [zdalny_plik]`
- `user [nazwa_konta]`

### pliki

#### /etc/vsftpd.conf

```bash
anonymous_enable=YES
listen=YES
listen_port=222
local_enable=YES
anon_world_readable_only=YES
local_umask=024
anon_umask=024
download_enable=YES
max_per_ip=2
max_clients=5
local_max_rate=655360
anon_max_rate=655360
ftpd_banner=Witaj
write_enable=YES
anon_upload_enable=NO
anon_mkdir_write_enable=NO
anon_other_write_enable=NO
no_anon_password=YES
```

## SSH

> sprawdzamy czy w /etc/resolv.conf jest prawidłowy adres servera na obu maszynach
---
>**instalacja**: apt install openssh-server openssh-client
>**Klient:** apt install openssh-client

### ważne pliki

- /etc/ssh/
- ~/.ssh/

### ważne komendy

- `ssh-keygen -t rsa`
- będąc w '~' `scp .ssh/id_rsa.pub student@192.168.0.1:` lub
- `ssh-copy-id -i /home/student/.ssh/id_rsa.pub student@192.168.0.1`
- `ssh student@192.168.0.1`
- `eval $(ssh-agent -s)` - usuwamy potrzebę wpisywania hasła `ssh ...`**
- `mkdir .ssh`, `cp id_rsa.pub .ssh/autorized_keys`

#### kopiowanie

- z lokalnej na zdalną
  - `scp [sciezka z plikiem] [nazwaUsera]@[IP_servera/Domena]:`
  - `scp file.txt student@192.168.0.1:`
- z zdalnej na lokalną
  - `scp [nazwaUsera]@[IP_servera/Domena]:[ścieżka pliku] [ścieżka lokalna]`
  - `scp student@192.168.0.1:file.txt ./`

---

## WWW

>**instalacja**: apt install apache2 apache2-utils apache2-mpm-prefork php5 php5-common mysql-server mysql-common libapache2-mod-php5 php5-mysql phpmyadmin

**W folderze /var/www tworzymy katalogi a w nich pojedyncze pliki do stron** `var/www/a/index.html` itd

### ważne pliki

- /etc/bind/db.ring
- /etc/apache2/apache2.conf
- /etc/apache2/sites-enabled/...

### ważne komendy

- `systemctl restart bind9`
- `systemctl restart apache2`

### pliki

#### do `/etc/bind/db.ring` dodajemy rekordy cname

```bash
#folder_strony      IN      CNAME       ring.hf.
a                   IN      CNAME       ring.hf.
b                   IN      CNAME       ring.hf.
##...
```

#### /etc/apache2/sites-enabled/000-default.conf

```xml
NameVirtualHost 192.168.0.1

<VirtualHost 192.168.0.1:80>
    ServerName a.ring.hf
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/a
</VirtualHost>

<VirtualHost 192.168.0.1:80>
    ServerName b.ring.hf
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/b
</VirtualHost>

<VirtualHost 192.168.0.1:8080>
    ServerName c.ring.hf
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/c
</VirtualHost>

<VirtualHost 192.168.0.1:443>
    ServerName d.ring.hf
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/d
    SSLEngine on
    SSLCertificateFile /etc/apache2/server.crt
    SSLCertificateKeyFile /etc/apache2/server.key

    <Directory /var/www/d>
        SSLRequireSSL
    </Directory>

</VirtualHost>
```

#### /etc/apache2/apache2.conf

```xml
<!-- możliwe że powinno to być w 000... w virtual server */b -->
<Directory /var/www/b>
    AuthType Basic
    AuthName "prosze podac haslo"
    AuthUserFile /var/www/.haslo
    Require valid-user
</Directory>
```

#### tworzenie uwierzytelniania

- `touch /var/www/.hasla`
- `htpasswd -c /var/www/.hasla ola`