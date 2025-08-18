Extplorer is an intermediate Proving Grounds box, also rated as intermediate by the community. This box exploits the `disk` group once we gain the initial foothold. It leads to accessing contents of `/etc/shadow` as `root` where we can escalate privileges.

`nmap` scan:

```
nmap <target ip> -sS -sC -sV -Pn -p-
```

![nmap-1](assets/extplorer/nmap-1.png)

Port `80` is open so we check that.

![port_80-1](assets/extplorer/port_80-1.png)

Seems like the website is using WordPress, could use `wpscan` to find potential vulnerabilities.

Let's `dirsearch` on this `url`:

```
python3 dirsearch.py -u <target ip> -e html,php -x 400,401,403,404
```

![dirsearch-1](assets/extplorer/dirsearch-1.png)

Check out `/filemanager` first:

![filemanager-1](assets/extplorer/filemanager-1.png)

A login field... Can we use default credentials (`admin`:`admin`)?

![login-1](assets/extplorer/login-1.png)

Nice, we can!

![extplorer-1](assets/extplorer/extplorer-1.png)

After clicking around a bit, it seems like under the `filemanager` folder there's another folder called `config` which seems interesting. Inside of `.htusers.php` we have what looks like 2 pairs of username and hashed password credentials. We know that the password for `admin` is `admin` since we used it to log in, so let's take a look at `dora`. Store the password into a file:

![password-1](assets/extplorer/password-1.png)

Any password cracker should work. We'll use `john` this time:

```
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john-1](assets/extplorer/john-1.png)

Maybe we have a password for `dora`? Try `ssh`:

![ssh-1](assets/extplorer/ssh-1.png)

Seems like we need a private key to log in as `dora`. Try other avenues, maybe we can upload files on `extplorer`:

![extplorer-2](assets/extplorer/extplorer-2.png)

This icon looks like it has upload functionality. Upload a basic `php` reverse shell from `pentestmonkey`:

![upload-1](assets/extplorer/upload-1.png)

![extplorer-3](assets/extplorer/extplorer-3.png)

Have a `nc` listener on port `80`, and then trigger the file with this command:

```
curl http://<target ip>/shell.php
```

Side note: the `url` is actually `http://<target ip>/filemanager/shell.php`, but by using this instead we error out:

![curl-1](assets/extplorer/curl-1.png)

So by omitting it we get this:

![shell-1](assets/extplorer/shell-1.png)

Awesome, got a shell!

We've gained an initial foothold, so it's possible that we can switch to `dora` using the password we've cracked before.

![dora-1](assets/extplorer/dora-1.png)

After grabbing `local.txt` let's transfer `linpeas` over and find anything interesting:

![linpeas-1](assets/extplorer/linpeas-1.png)

`dora` being a part of the `disk` group is incredibly useful. We can run `df -h` and have a deeper look into the filesystem of the machine:

![df_h-1](assets/extplorer/df_h-1.png)

Windows works with physical HDDs and SSDs like `C:\` and `D:\`. In linux, it works a little differently: data is stored in virtual block devices, and that's what each entry here represents. So, `/dev/mapper/ubuntu--vg-ubuntu--lv` is a block device mounted at the root `/` of the machine. Now, as a member of the `disk` group we're able to bypass any permissions if we have direct access to data. So as `dora` we can run `debugfs` which can output contents of any folder or file, including incredibly sensitive ones.

```
debugfs -w /dev/mapper/ubuntu--vg-ubuntu--lv
```

After entering "debug" mode we can extract contents from `/etc/shadow`:

![shadow-1](assets/extplorer/shadow-1.png)

Store this hash in `hash.txt` and crack it with `john` like before:

![john-2](assets/extplorer/john-2.png)

Switch to `root` with `su root` and type in `explorer`:

![root-1](assets/extplorer/root-1.png)

Rooted! :partying_face:
