
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
ssh-copy-id -i $HOME/.ssh/<keyname>.pub cardano@server.public.ip.address
```

Login with your new cardano user

```
ssh cardano@server.public.ip.address
```

Disable root login and password based login. Edit the `/etc/ssh/sshd_config file`

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

Locate **PermitRootLogin** and update to prohibit-password

```
PermitRootLogin prohibit-password
```

Locate **PermitEmptyPasswords** and update to no

```
PermitEmptyPasswords no
```

**Optional**: Locate Port **and customize it to your random port number**.

{% hint style="info" %}
Use a **random** port # from 1024 thru 49141. [Check for possible conflicts.](https://en.wikipedia.org/wiki/List\_of\_TCP\_and\_UDP\_port\_numbers)
{% endhint %}

```bash
Port <port number>
```

Validate the syntax of your new SSH configuration.

```
sudo sshd -t
```

If no errors with the syntax validation, restart the SSH process.

```
sudo systemctl restart sshd
```

Verify the login still works

{% tabs %}
{% tab title="Standard SSH Port 22" %}
```
ssh cardano@server.public.ip.address
```
{% endtab %}

{% tab title="Custom SSH Port" %}
```bash
ssh cardano@server.public.ip.address -p <custom port number>
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Alternatively, add the `-p <port#>` flag if you used a custom SSH port.

```bash
ssh -i <path to your SSH_key_name.pub> cardano@server.public.ip.address
```
{% endhint %}

**Optional**: Make logging in easier by updating your local ssh config.

To simplify the ssh command needed to log in to your server, consider updating your local `$HOME/.ssh/config` file:

```bash
Host cardano-server
  User cardano
  HostName <server.public.ip.address>
  Port <custom port number>
```

This will allow you to log in with `ssh cardano-server` rather than needing to pass through all ssh parameters explicitly.

## :robot: **Updating Your System**

{% hint style="warning" %}
It's critically important to keep your system up-to-date with the latest patches to prevent intruders from accessing your system.
{% endhint %}

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get autoremove
sudo apt-get autoclean
```


# Disable firewall
sudo ufw disable
```


