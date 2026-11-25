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

## Scenario 2: Port Conflict

### Symptom
Attempting to start a temporary Python HTTP server on port `8080` and a second instance on port `8081` failed with:
```
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/usr/lib/python3.12/http/server.py", line 1314, in <module>
    test(
  File "/usr/lib/python3.12/http/server.py", line 1261, in test
    with ServerClass(addr, HandlerClass) as httpd:
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/socketserver.py", line 457, in __init__
    self.server_bind()
  File "/usr/lib/python3.12/http/server.py", line 1308, in server_bind
    return super().server_bind()
           ^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/http/server.py", line 136, in server_bind
    socketserver.TCPServer.server_bind(self)
  File "/usr/lib/python3.12/socketserver.py", line 473, in server_bind
    self.socket.bind(self.server_address)
OSError: [Errno 98] Address already in use
```
### Diagnostic Commands

Checked port 8080 with `ss -tulnp | grep 8080`
```
tcp   LISTEN 0      511                                      0.0.0.0:8080       0.0.0.0:*
```
Because the socket was owned by another user, sudo was required to reveal the PID:
```
$ sudo ss -tulnp | grep 8080
tcp   LISTEN 0      511                                      0.0.0.0:8080       0.0.0.0:*    users:(("node",pid=1678,fd=24))
```
Inspected PID 1678 with `ps -fp 1678`:
```
UID          PID    PPID  C STIME TTY          TIME CMD
webapp-+    1678       1  0 12:50 ?        00:00:00 /usr/bin/node /opt/simple-app/server.js
```
Since port `8080` was held by an existing `systemd`-managed service (`PPID 1`), the test was moved to port `8081`. After starting one `python3 -m http.server 8081` and triggering the conflict with a second instance, ran `ss -tulnp | grep 8081`:

```
tcp   LISTEN 0      5                                        0.0.0.0:8081       0.0.0.0:*    users:(("python3",pid=4749,fd=3))
```
Confirmed the command behind PID 4749 using `ps -fp 4749`.

### Root Cause
Only one process can bind to a given TCP port at a time. Port 8080 was already bound by `/usr/bin/node /opt/simple-app/server.js` (PID 1678), and port 8081 was bound by the first python3 -m http.server 8081 process (PID 4749).

### Fix & Verification
Left the systemd-managed simple-app service untouched on port 8080, and cleanly terminated the conflicting test server on port 8081 using SIGTERM (allowing graceful socket cleanup before resorting to kill -9):
`kill 4749`
Verified the port was released by running `ss -tulnp | grep 8081`, which returned empty.


## Scenario 3: Runaway CPU Process

### Symptom
System CPU usage spiked due to a background process consuming near-100% of a CPU core.

### Diagnostic Commands

Started a safe simulated CPU-intensive process in the background:
`yes > /dev/null &`
Identified the top CPU-consuming processes with `ps aux --sort=-%cpu | head -n 5`:
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
user        8704 99.9  0.0   8300  2044 pts/3    R    18:21   1:01 yes
user        8935  1.9  0.4 106012 15648 ?        S    18:22   0:00 /usr/lib/git-core/git-remote-https origin https://github.com/dinith01/linux-host-telemetry-engine.git
mysql       1751  1.2 11.9 2376324 394576 ?      Ssl  12:50   4:11 /usr/sbin/mysqld
user        7374  0.5 14.9 19343376 490656 ?     Sl   16:18   0:44 /home/user/.vscode-server/bin/69047191fc427f2f92828279aae1d90d8eeda7ab/node --dns-result-order=ipv4first /home/user/.vscode-server/bin/69047191fc427f2f92828279aae1d90d8eeda7ab/out/bootstrap-fork --type=extensionHost --transformURIs --useHostProxy=false
```

### Root Cause

An uncontrolled yes loop (PID-8704) was continuously writing to `/dev/null`, saturating a CPU core.


### Fix & Verification
Terminated the runaway process gracefully using `SIGTERM` (`kill 8704` rather than `kill -9`, so the process can clean up before exiting):
`kill 8704`
Verified the process was no longer running with 
```
ps aux | grep yes
ps aux | grep yes
user        9104  0.0  0.0   9156  2320 pts/3    S+   18:23   0:00 grep --color=auto yes
```