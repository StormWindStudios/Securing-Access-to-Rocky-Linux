## Securing Access to Rocky Linux

### 1. Hostname, Time Zone, and Updates ###
Logging out and then back in isn't necessary, but it will update your hostname and potentially save space in the CLI.
```
[root@rockylinux-s-2vcpu-4gb-amd-sfo3-01 ~]# hostname rocky1
[root@rockylinux-s-2vcpu-4gb-amd-sfo3-01 ~]# exit
```

List the available timezones and choose your own. Also, verify your system time is synchronized and correct (this will be important later).
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
Run Updates
```
[root@rocky1 ~]# dnf update -y
```

### 2. Configure a New User ###


