## Securing Access to Rocky Linux

### 1. Hostname, Time Zone, and Updates ###
Use the `hostname` command to change your system's name. Logging out and then back in isn't necessary, but it will update your hostname and potentially save space in the CLI.
```
[root@rockylinux-s-2vcpu-4gb-amd-sfo3-01 ~]# hostname rocky1
[root@rockylinux-s-2vcpu-4gb-amd-sfo3-01 ~]# exit
```

Use `timedatectl` to the available timezones and set your own, and verify your system time is synchronized (this will be important later).
```
[root@rocky1 ~]# timedatectl list-timezones
--- Output Omitted ---
[root@rocky1 ~]# timedatectl set-timezone America/Phoenix
[root@rocky1 ~]# timedatectl
               Local time: Fri 2022-02-11 17:07:05 MST
           Universal time: Sat 2022-02-12 00:07:05 UTC
                 RTC time: Sat 2022-02-12 00:07:04
                Time zone: America/Phoenix (MST, -0700)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```
Run updates using `dnf`.
```
[root@rocky1 ~]# dnf update -y
--- Output Omitted ---
```

### 2. Configure a New User ###

Create a new user and set its password.
```
[root@rocky1 ~]# adduser shane
[root@rocky1 ~]# passwd shane
Changing password for user shane.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
```

Give the account sudo (administrative) privileges by adding it to the `wheel` group.
```
[root@rocky1 ~]# usermod -aG wheel shane
```

Now, create a new `.ssh` directory in your user's home, copy the SSH key information to it, and update ownership.
```
[root@rocky1 ~]# mkdir /home/shane/.ssh
[root@rocky1 ~]# cp ~/.ssh/authorized_keys /home/shane/.ssh/
[root@rocky1 ~]# chown -R shane:shane /home/shane/.ssh
```

Verify you are able to log in as the new user in a new SSH session. Also, verify you can use `sudo` as below:
```
[shane@rocky1 ~]$ sudo ls

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for shane: 
[shane@rocky1 ~]$
```
### 3. Harden SSH Configuration ###
Return to the root SSH session for now. Run `vi /etc/ssh/sshd_config` and make the following updates:
* `Port 2358` (or another port number of your choice)
* `LoginGraceTime 1m`
* `PermitRootLogin no`
* `MaxAuthTries 3`
* `MaxSession 5`
* `PubKeyAuthentication yes`
* `PasswordAuthentication no`
* `IgnoreRhosts yes`

Save the file, inform SELinux of the change (remember to use the same port number), and reload sshd.
```
[root@rocky1 ~]# semanage port -a -t ssh_port_t -p tcp 2358
[root@rocky1 ~]# systemctl reload sshd
```

From yet another SSH session, verify you can login as your user but *not* as root. Remember to point SSH at the right port!

### 4. Install and Configure Firewalld ###

Install and configure firewalld to run automatically.
```
[root@rocky1 ~]# dnf install firewalld -y
[root@rocky1 ~]# systemctl start firewalld
[root@rocky1 ~]# systemctl enable firewalld
```

Disable the SSH service (since we're no longer using port 22) and open your new SSH port. Reload the firewall for the changes to take effect.
```
[root@rocky1 ~]# firewall-cmd --remove-service ssh --permanent
success
[root@rocky1 ~]# firewall-cmd --add-port 2358/tcp --permanent
success
[root@rocky1 ~]# systemctl reload firewalld
```

Check that you can still log in with your non-root user. Make sure you leave open sessions until you're sure the firewall is configured correctly!

### Configure Google Authenticator
Install the Google Authenticator app on your phone. Then, **switch over to your non-root user** and install it on your Linux server.
```
[shane@rocky1 ~]$ sudo dnf install -y epel-release
[shane@rocky1 ~]$ sudo dnf install -y epel-release google-authenticator qrencode qrencode-libs
```
Run Google Authenticator from your user account and configure it. For all yes or no questions, choose **y**. When prompted, scan the QR code with the Google Authenticator app on your phone. Enter the code generated on your phone.

Now, we need to tell Linux to use Google Authenticator. run `sudo vi /etc/pam.d/sshd`. Comment out the topmost "password auth" line, and add the Google Authenticator line below it.
```
#auth      substack     password-auth
auth       required     pam_google_authenticator.so secret=/home/${USER}/.ssh/.google_authenticator
```
Save and exit. 

Now run `sudo vi /etc/ssh/sshd_config`. Ensure the line `ChallengeResponseAuthentication yes` is present and uncommented. At the bottom, add the line `AuthenticationMethods publickey,keyboard-interactive` to the bottom. Save and exit.

Move the Google Authenticator file to the `.ssh` directory, where SELinux will allow sshd to access it. Finally, reload sshd.
```
[shane@rocky1 ~]$ mv ~/.google_authenticator ~/.ssh/
[shane@rocky1 ~]$ sudo systemctl reload sshd
```

If everything is configured correctly, you will now log in using two authentication mechanisms: your private key and the Google Authenticator code.

