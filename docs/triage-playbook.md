# Linux Incident Triage Playbook

## Scenario 1: Broken Service Configuration

### Symptom
Running `sudo systemctl restart nginx` failed with a non-zero exit status:
```text
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```
### Diagnostic Commands
Running `sudo journalctl -u nginx -n 20 --no-pager`  outputs an error on line 2
```
Nov 20 19:54:34 user-VirtualBox systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Nov 20 19:54:34 user-VirtualBox systemd[1]: nginx.service: Failed with result 'exit-code'.
Nov 20 19:54:34 user-VirtualBox systemd[1]: Failed to start nginx.service - A high performance web server and a reverse proxy server.
Nov 20 20:35:28 user-VirtualBox systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 20 20:35:29 user-VirtualBox nginx[4378]: 2026/11/20 20:35:29 [emerg] 4378#4378: invalid number of arguments in "user" directive in /etc/nginx/nginx.conf:2
Nov 20 20:35:29 user-VirtualBox nginx[4378]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 20 20:35:29 user-VirtualBox systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Nov 20 20:35:29 user-VirtualBox systemd[1]: nginx.service: Failed with result 'exit-code'.
Nov 20 20:35:29 user-VirtualBox systemd[1]: Failed to start nginx.service - A high performance web server and a reverse proxy server.
Nov 20 20:47:29 user-VirtualBox systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 20 20:47:30 user-VirtualBox systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
Nov 20 20:54:31 user-VirtualBox systemd[1]: Stopping nginx.service - A high performance web server and a reverse proxy server...
Nov 20 20:54:31 user-VirtualBox systemd[1]: nginx.service: Deactivated successfully.
Nov 20 20:54:31 user-VirtualBox systemd[1]: Stopped nginx.service - A high performance web server and a reverse proxy server.
Nov 20 20:54:31 user-VirtualBox systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 20 20:54:31 user-VirtualBox nginx[5989]: 2026/11/20 20:54:31 [emerg] 5989#5989: invalid number of arguments in "user" directive in /etc/nginx/nginx.conf:2
Nov 20 20:54:31 user-VirtualBox nginx[5989]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 20 20:54:31 user-VirtualBox systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Nov 20 20:54:31 user-VirtualBox systemd[1]: nginx.service: Failed with result 'exit-code'.
Nov 20 20:54:31 user-VirtualBox systemd[1]: Failed to start nginx.service - A high performance web server and a reverse proxy server.
```

Tested the Nginx configuration files using the `sudo nginx -t` command:
```
2026/11/20 21:03:35 [emerg] 6524#6524: invalid number of arguments in "user" directive in /etc/nginx/nginx.conf:2
nginx: configuration file /etc/nginx/nginx.conf test failed
```

### Root Cause
Missing a semicolon in `/etc/nginx/nginx.conf` on line 1

### Fix & Verification
Added the semicolon to the end of this line:
``` 
user www-data;
```
Then ran:
```
sudo systemctl restart nginx
sudo systemctl status nginx
```
This is the running output:
```
nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-11-20 20:47:30 +0530; 2s ago
       Docs: man:nginx(8)
    Process: 5643 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 5645 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 5646 (nginx)
      Tasks: 6 (limit: 2810)
     Memory: 6.6M (peak: 6.6M)
        CPU: 44ms
     CGroup: /system.slice/nginx.service
             ├─5646 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─5647 "nginx: worker process"
```
Ran `sudo systemctl is-active nginx`, which returned `active`.