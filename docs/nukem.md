Nukem is an intermediate Proving Grounds box, also rated as hard by the community. In this machine, we use `wp-scan` to find vulnerabilities within the WordPress version of the service. Then we escalate privileges through SUID binary exploitation.

`nmap` scan:

![nmap-1](assets/nukem/nmap-1.png)

We see ports `80` and `5000` open, both running `http`. We also discover that port `80` is running on WordPress 5.5.1 which could be useful later. Let's check out port `80`:

![port_80-1](assets/nukem/port_80-1.png)

Seems like a basic WordPress site, nothing too exciting. We can try `dirsearch` on this:

```
python3 dirsearch.py -u http://<target ip> -e html,php,xml -x 400,401,403,404
```

![dirsearch-1](assets/nukem/dirsearch-1.png)

We'll check these paths out soon. Enumerate WordPress 5.5.1 through `searchsploit`:

![searchsploit-1](assets/nukem/searchsploit-1.png)

Alright so we got a matching version number on `searchsploit` and it tells us that it's SQL Injection. Download it, then `cat` out the contents:

```
searchsploit -m 42291
```

![exploit-1](assets/nukem/exploit-1.png)

`wpscan` can also be of use here since WordPress is running on the target:

```
wpscan --url http://<target ip>
```

![wpscan-1](assets/nukem/wpscan-1.png)

Getting into some issues here trying to run `wpscan`. Through some trial and error and Googling, we find several commands to install certain packages that'll help with running the scanner:

```
sudo apt update
sudo apt install ruby-full build-essential
sudo gem install bundler
sudo gem install wpscan
```

`ruby-full` installs everything needed to work with ruby, and `build essential` installs compiler tools.

Let's run `wpscan --url http://<target ip>` once more:

![wpscan-2](assets/nukem/wpscan-2.png)

![wpscan-3](assets/nukem/wpscan-3.png)

![wpscan-4](assets/nukem/wpscan-4.png)

`wpscan` was a success this time, and based on our `dirsearch` results we find a familiar path: `/uploads`. Couldn't hurt to check it out:

![uploads-1](assets/nukem/uploads-1.png)

`simple-file-list/` looks interesting:

![simple_file_list-1](assets/nukem/simple_file_list-1.png)

"Simple File List" is a familiar service found in our `wpscan`:

![wpscan-5](assets/nukem/wpscan-5.png)

This service could have vulnerabilities. Let's enumerate with `searchsploit`:

![searchsploit-2](assets/nukem/searchsploit-2.png)

We can go on `exploit-db` and try either the Arbitrary File Upload or the RCE exploits. We'll go with the File Upload first:

[Exploit DB](https://www.exploit-db.com/exploits/48979)

![exploit_db-1](assets/nukem/exploit_db-1.png)

Can change this payload to our liking after downloading the exploit, so to test we try this:

![php-1](assets/nukem/php-1.png)

Execute the exploit:

```
python3 48979.py http://<target ip>
```

![exploit-2](assets/nukem/exploit-2.png)

It says that the file was moved to `http://<target ip>/wp-content/uploads/simple-file-list/8447.php`, we can go and visit the file in the browser:

![exploit-3](assets/nukem/exploit-3.png)

Awesome, our basic `cmd` command in the payload executed. This means we can try going for a reverse shell. From `pentestmonkey` we can download a simple `php` reverse shell script and then paste the entire thing into the payload of the exploit:

![reverse_shell-1](assets/nukem/reverse_shell-1.png)

Set up a listener on port `80` and run the exploit once more:

![shell-1](assets/nukem/shell-1.png)

Cool, our initial foothold. Check out `srv`:

![srv-1](assets/nukem/srv-1.png)

We see `wp-config.php` which seems important so let's `cat` it out:

![config-1](assets/nukem/config-1.png)

![config-2](assets/nukem/config-2.png)

![config-3](assets/nukem/config-3.png)

`DB_USER` is `commander`, and looking around the machine we find that `commander` is also a user on the target. We have `DB_PASSWORD` as well (`CommanderKeenVorticons1990`), so maybe the passwords match both on their database and the target machine since password reuse is a common issue.

```
su commander
```

Then we paste in `CommanderKeenVorticons1990` as the password:

![commander-1](assets/nukem/commander-1.png)

`sudo -l` to find any permissible `sudo` commands:

![sudo_l-1](assets/nukem/sudo_l-1.png)

Nothing. Try finding any interesting `SUID` binaries:

```
find / -type f -perm -04000 2>/dev/null
```

![suid-1](assets/nukem/suid-1.png)

Out of the usual `SUID` subjects, `dosbox` sticks out as an irregular. `GTFOBins` tells us more:

![gtfobins-1](assets/nukem/gtfobins-1.png)

Through `dosbox` we can write anything to any system file, so by giving `commander` all possible commands without a password inside of `/etc/sudoers`, we would have effectively created a `root` user, and all we would need to do is just switch to `root` to root the target.

![exploit-4](assets/nukem/exploit-4.png)

Switch to `root` with:

```
sudo su root
```

Rooted! :partying_face:
