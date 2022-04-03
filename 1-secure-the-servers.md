
# Hardening an Ubuntu Server


## Creating a Non-root User with sudo Privileges


Make a habit of logging to your server using a non-root account. This will prevent the accidental deletion of files if you make a mistake. For instance, the command rm can wipe your entire server if run incorrectly using by a root user.



:fire: **Tip**: Do NOT routinely use the root account. Use `su` or `sudo`, always.


SSH to your server using MobaXterm


Create a new user called cardano

```
useradd -m -s /bin/bash cardano
```

Set the password for cardano user

```
passwd cardano
```

Add cardano to the sudo group

```
usermod -aG sudo cardano
```

**Now you should access the servers with MobaXterm, by editing the username to cardano**

## :lock\_with\_ink\_pen: **Disabling SSH Password Authentication and Using SSH Keys Only**


The basic rules of hardening SSH are:

* No password for SSH access (use private key)
* Don't allow root to SSH (the appropriate users should SSH in, then `su` or `sudo`)
* Use `sudo` for users so commands are logged
* Log unauthorized login attempts (and consider software to block/ban users who try to access your server too many times, like fail2ban)
* Lock down SSH to only the ip range your require (if you feel like it)


Create a new SSH key pair on your local machine. Run this on your local machine. You will be asked to type a file name in which to save the key. This will be your **keyname**.

Open PowerShell in Windows and run the following:
```
ssh-keygen -t ed25519
```

In the end two files should be created inside C:\Users\ **your_windows_user**

Open the file you created that ends as .pub with a notepad

Now back to the server, run the following commands:

```bash
mkdir .ssh
cd .ssh
nano authorized_keys
```
Copy the contents of the .pub file you opened before into the nano text editor inside the server and save the file by pressing CTRL + X, pressing Y and then Enter

Close the MobaXterm session for that server and edit the settings to connect with the keyfile you created inside C:\Users\ **your_windows_user** . You need to use the file that doesn't have an extension (the one that doesn't end as .pub).

## Disable firewall
```
sudo ufw disable
```

## Disable root login and password based login. Edit the `/etc/ssh/sshd_config file` ##

```
sudo nano /etc/ssh/sshd_config
```

Locate **ChallengeResponseAuthentication** and update to no

```
ChallengeResponseAuthentication no
```

Locate **PasswordAuthentication** update to no

```
PasswordAuthentication no 
```

Locate **PermitRootLogin** and update to no

```
PermitRootLogin prohibit-password
```

Locate **PermitEmptyPasswords** and update to no

```
PermitEmptyPasswords no
```
Locate **Port** and customize it to 5672

```bash
Port 5672
```

In the end save the file by pressing CTRL + X, pressing Y and then Enter


Validate the syntax of your new SSH configuration.

```
sudo sshd -t
```

If no errors with the syntax validation, restart the SSH process.

```
sudo systemctl restart sshd
```

Verify the login still works by attempting to connect with MobaXterm using port 5672
