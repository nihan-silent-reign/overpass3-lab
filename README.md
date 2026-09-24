# overpass3-lab

Overpass 3 TryHackMe Write up A detailed walkthrough of the Overpass 3 room on TryHackMe, demonstrating enumeration, GPG decryption, exploiting FTP for a reverse shell, and leveraging an NFS no_root_squash misconfiguration for local privilege escalation.


Phase 1: Enumeration & Reconnaissance1. 

Port Scanning The assessment began with an Nmap scan to identify open ports and active services on the target system.

The scan revealed three open ports:

Port 22: SSH Port
Port 21: FTP Port 
Port 80: HTTP (Apache httpd)


2. Directory Brute-Forcing

While reviewing the source code of the main webpage did not reveal any anomalies, a directory brute-forcing tool (Gobuster) uncovered a hidden directory: /backup.




Phase 2: Gaining Access1. Decrypting the Backup
After downloading and unzipping backup.zip, it contained two primary files:

A GPG-encrypted file.
A GPG private key file.

To access the encrypted data, the private key was imported into GPG, and the file was successfully decrypted:

command used:
gpg --import private.key
gpg --decrypt encrypted_file.gpg > customer.xlsx


Opening customer.xlsx in an online spreadsheet viewer revealed a table containing usernames and passwords for three distinct users.



2. Initial Foothill via FTP

Using one of the sets of credentials obtained from the spreadsheet, an active session was established via FTP.

Because the FTP root directory mapped directly to a web-accessible area, initial access could be upgraded to a reverse shell. 

A standard Metasploit PHP reverse shell payload was uploaded to the server via FTP.

A Meterpreter listener was configured on the attacker machine, the uploaded PHP script was executed via the browser, and a successful reverse shell connection was caught.




Phase 3: Privilege Escalation

1. Local Enumeration

To find a path for privilege escalation, the automated enumeration script linpeas.sh was uploaded to the target's /tmp directory and executed.

The scan highlighted two major internal configurations:

Port 2049 (NFS) was active internally.

An NFS share belonging to the user james was configured with no_root_squash





2. Exploiting NFS no_root_squash

The no_root_squash option is a critical misconfiguration. By default, NFS squashes root permissions from a client to a low-privilege user (nobody) on the server. When no_root_squash is enabled, if a remote client connects to the share as root, they retain root privileges over the files within that share.

Since the NFS port was only listening internally, an SSH port forward was established using james's newly discovered SSH private key (id_rsa) found in his directory.

Once the port was forwarded to the attacker machine, the remote share was mounted locally:
commands used:
sudo mount -t nfs -o port=<FORWARDED_PORT> localhost:/home/james /mnt/target_james

This granted full access to James's home directory, allowing the retrieval of the user flag.




3. Escalating to Root

To upgrade privileges from the user james to full system root:

On the attacker machine (acting as local root), a copy of the /bin/bash binary was transferred into the mounted NFS directory.

The owner of the copied binary was changed to root.

The SUID bit was set on the binary:
comand used:
sudo chown root:root /mnt/target_james/bash
sudo chmod +s /mnt/target_james/bash




4.Returning to the SSH session as the user james, the SUID binary was executed:
command used:
./bash -p




This successfully dropped the shell into a root environment, granting full control over the target system and allowing the collection of the final root and web flag.




