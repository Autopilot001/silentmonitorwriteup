# Silent Monitor my room notes

**Author:** Mohamed Nabil Algamoudi  
**Room:** Lab Instance  Silent Monitor (Junior Penetration Testing)

These are my notes from completing the room. I used the TryHackMe target and AttackBox/VPN environment. I replaced the assigned target address with **targetip** so these commands can be read without publishing the address from my session. I have not included any flag values or passwords.

## 1. Start the target and connect

I started the target machine and AttackBox, connected to the TryHackMe VPN, and copied the assigned address into my notes. Before running a command, I replaced **target ip** below with the address for that active room instance.

![Silent Monitor portal landing page](../screenshots/step-01-portal.png)

## 2. Enumerate the target

I first scanned all TCP ports, then checked the detected services and default scripts:

~~~bash
sudo nmap -sC -sV -p- -Pn TARGET_IP
~~~

The scan showed SSH on port 22 and a Werkzeug web server on port 5050. The HTTP title identified the site as the CorpNet Network Operations Centre.

I enumerated web paths on port 5050:

~~~bash
gobuster dir -u http://TARGET_IP:5050 -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
~~~

The scan found /internal with a 200 response.

## 3. Check the portal login

I opened the portal and checked the login behavior. An invalid attempt returned an “Invalid username or password” message. I started down a Hydra route, but that was unnecessary. Note to myself: test the application behavior before spending time on password enumeration.

![Portal sign-in page](../screenshots/step-02-login.png)

![Portal response to an invalid login](../screenshots/step-03-login-response.png)

After inspecting the login behavior, I tested this input:

~~~text
admin' OR 1=1--
~~~

The login accepted it. That gave me the SQL injection foothold. I then reviewed the authenticated portal and its audit information.

![Authenticated portal audit view with network values redacted](../screenshots/step-06-portal-audit.png)

## 4. Follow the health-check clue

After logging in, I checked /internal/health and the application logs. One log entry contained an encoded newline followed by whoami:

~~~text
127.0.0.1%0awhoami
~~~

The input did not behave the way I expected through the page. I intercepted and replayed the request in Burp Suite. I tested the health-check input with a ping version check:

~~~text
ping -c 2 -W 1 127.0.0.1 -V
~~~

The response showed the iputils version. This confirmed that the health-check input reached command execution. Note to myself: compare the raw request and response when browser behavior is unclear.

![Burp health-check response; request details are redacted](../screenshots/step-04-health-check.png)

## 5. Inspect the application files

I used the healthcheck behavior to inspect the application files. I found app.py and secret.config. The Python file was the backend; the configuration exposed the backupagent account and showed that it ran as sysadmin. It also contained a password, which I have not copied into these notes.

![Application source inspection; request details are redacted](../screenshots/step-05-source-inspection.png)

## 6. Use the recovered account over SSH

I tried the recovered sysadmin credentials against SSH:

~~~bash
ssh sysadmin@TARGET_IP
~~~

SSH accepted them and gave me a shell on the target. I used the address and hostkey prompt shown by my active room instance.

## 7. Check permissions and backups

I checked the account’s sudo permissions:

~~~bash
sudo -l
~~~

The account could not run sudo. I listed the home directory and found backups and user.txt. Inside backups were README.txt and infrastructure.kdbx.

## 8. Work through the KeePass database

I used keepass4crack.py (credited to unix-ninja) to recover the KeePass database password. I have removed the recovered password and account details from the public screenshot and left the password out of these notes.

![KeePass recovery output with credential details redacted](../screenshots/step-07-keepass.png)

Using the database information, I completed the final escalation, obtained root access, and checked /root/root.txt to confirm completion. I have not included any flag values.

![Root flag file check; terminal identity is redacted and no flag value is shown](../screenshots/step-08-root-check.png)

## 9. Review HTTP objects in Wireshark

I used Wireshark’s **File → Export Objects → HTTP** menu while reviewing the captured traffic. The screenshot keeps the menu path visible and covers the packet details that could identify addresses.

![Wireshark HTTP object export menu](../screenshots/step-09-wireshark-export.png)

## Notes to myself

Skip Hydra for this room the login input test exposed the useful weakness sooner.
Keep the Burp request and response when browser behavior is unclear.
Treat configuration files and backup databases as sensitive. Do not copy recovered secrets into a public writeup.
Replace **TARGET_IP** with the address assigned to the current Lab Instance 

## Remediation notes

The login should use parameterized queries. The health-check should not pass user input to a shell. Service credentials should be kept out of configuration files, rotated if exposed, and restricted to the access the account needs. Backup databases should be protected with strong, unique credentials.
