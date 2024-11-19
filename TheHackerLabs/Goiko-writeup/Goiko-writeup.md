# User flag

I began by running an Nmap scan on the victim machine to identify the open ports.  
I discovered that the accessible ports are 22, 139, 445, and 10021.

![nmap](nmap.png)

Since the SSH version configured on port 22 is not vulnerable, I moved on to checking ports 139 and 445, which are associated with SMB.

To review the SMB content, I used `smbclient` without specifying a user (`-N`) and listing the content (`-L`).

![](smb-list.png)

In the output of the command, I received several uncommon "share names," so I proceeded to list the content of one of them.

![](creds-food.png)

When listing the content of the `food` directory, I found a file with credentials. I performed a `get` to retrieve this file and inspect its contents, but upon doing so, I encountered the following message:

![](creds-trap.png)

As stated in the message, we will need to make more effort to obtain the credentials. Therefore, we listed the other two directories to attempt to find more useful information.

In the `dessert` directory, we found three files that we retrieved for review.

![](food-dir.png)

Upon reviewing the files, we observed some strange messages that did not reveal any sensitive information.

Continuing with the same process for the `menu` directory, we found a user in the `.cafesinleche` file.

![](menu-credentials.png)

We then attempted to log in via SSH using these credentials, and we succeeded.

![](access-gained.png)

For better convenience, it's recommended to upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Once we gained access, we proceeded with privilege escalation by attempting to access the FTP port, which we saw was open during the Nmap scan. There, using the `marmai` user credentials, we found a ZIP file called `BurgerWithoutCheese.zip`.

![](ftp-marmai.png)
 
However, when we tried to access its contents, we were asked for an `id_rsa` password, which we did not have.

Nevertheless, this doesn't mean we can't extract the contents of the ZIP file.

First, we need to convert the content to `john` format, so we used `zip2john` for this purpose. Then, we ran `john` on the file where we stored the content, and we found that the password is `princess95`.

![](zip-pass.png)

We extracted the content of the `BurgerWithoutCheese.zip` file, which gave us an `id_rsa` file and the following list of users:

```bash
nika                                                          caroline                                                      gurpreet
santi
marsu
```

To continue with privilege escalation, we tried to SSH into the machine using one of these users and the provided `id_rsa`.

When we attempted this, we found that we couldn't access the machine with the provided `id_rsa`, so we proceeded to crack it using `john` again to discover the password.

![](idrsa.png)

Next, we tried accessing with the different available users and succeeded with `gurpreet`, obtaining the user flag.

![](user-flag.png)


# Root flag

In the same directory where we found the user flag, we saw a file called `nota` with the following content:

```bash
- ENGLISH = The database has very simple hashes, please configure it well. 

- CASTELLANO = La base de datos tiene hashes muy sencillos, por favor configuralo bien.

- CATALA = La base de dades te hashes molt senzills, si us plau configura be.
```

Running `mysql --version`, we saw that a MariaDB database was installed, to which we could connect using the `gurpreet` user.

![](mariadb.png)

Searching in the `secta` database, we found the hashes for the `carline` and `nika` users.

![](bd-hashes.png)

Using `hash-identifier`, we found that the hashes were stored using MD5.

![](hash-identifier.png)

To crack the passwords, we saved both hashes to a file and used `Hashcat` with the following command:

```bash
hashcat -m 0 hashes /usr/share/wordlists/rockyou.txt
```

And we obtained the following passwords.

![](hashcat-result.png)

Once inside `nika`'s account, we ran `sudo -l` to check if we had permission to execute any file as root.

![](sudo-l.png)

As we can see in the script, there is a call to `find` without using absolute paths, which allows us to execute a **PATH HIJACKING** attack.

To perform this, we created a `find` file with the following content:
```bash
chmod u+s /bin/bash
```

By adding the `/tmp` directory, where we have our `find` file stored, to the `PATH`, the command inside the file will be executed with root privileges.

In this way, our user will have permission to run `/bin/bash` as root, allowing us to access the root flag.

![](root.png)

Finally, we would like to thank `Rev3rK1hll`, the creator of this machine, who has many other machines available on TheHackerLabs.