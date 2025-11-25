# RHCSA_Preparation



## Install RHEL

- using ISO, developer account subscription
- using Vagrant
 
## Connect to RHEL using SSH

In real time, remote login into the VM using the SSH. mostly using username and password, sometimes username and private key.

Full information: RHCSA_Preparation/ssh_to_VM
 

## Hypervisors

What is the Hypervisor?

Hypervisor is a software used to create and runs virtual machines on host machine. 

What Hypervisor will do?

Hypervisor abstracts the underliying resources of host machine and allows the host machine share hardware resources among the VMs.

Types of Hypervisors?

There are 2 types of hypervisors:

1. type 1 hypervisor (Bare metal or native)

    ------------------------
    | vm1 |  vm2  | ... vmn
    ------------------------
      hypervisor 
      (VMWare ESXi, Hyper-V)
    ------------------------
      Hardware resources
    ------------------------

2. type 2 hypervisor (Hosted Hypervisor)

    ------------------------
    | vm1 |  vm2  | ... vmn
    ------------------------
      hypervisor 
      (VMWare Workstation, 
      Oracle vm Virtual box)
    ------------------------
          Host OS
    ------------------------
      Hardware resources
    ------------------------

---

## Module 01 - linux Essentials

sudo useradd -m <username>

sudo passwd <username>

sudo usermod -aG <groupname> <username>
(or)
sudo groupmod -a -U <username> <group-name>


### About sudo

sudo -> used to elevated previliages of current user

sudo yum install -y bash-completion

Note:
The sudo command allows permitted users to execute commands as another user, typically root.

Configurations are managed in **/etc/sudoers or under /etc/sudoers.d/**.

You should never edit /etc/sudoers directly; instead use:

    sudo visudo
    
    
Edit /etc/sudoers.d/john to allow only specific commands:

    john ALL=(ALL) /bin/systemctl, /usr/bin/yum
    
    
How to check the groups the user belog to?

    sudo id -Gn user1

    or

    groups user1

    or

    id # current user full details
    

Question:
----------
Create a user named **devops** who can restart the httpd service using sudo but cannot run any other privileged command.

Create user

    sudo useradd -m devops

    sudo passwd devops

    # enter the password

Check the httpd service is installed

    sudo systemctl status httpd || sudo yum install -y httpd
    
    sudo systemctl status httpd
    
    sudo systemctl start httpd
    
    sudo systemctl enable httpd
    
    Check the user and group:

    -rw-r--r--.  1 root root   963 Jul 28 18:24 httpd.service
    
Create a sudoers policy file for the user

    We never edit /etc/sudoers directly — always use visudo or a file under /etc/sudoers.d/
    
    sudo visudo -f /etc/sudoers.d/devops
        
        devops ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd, /bin/systemctl status httpd
        
        # devops ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd.service, /bin/systemctl status httpd.service
    
    sudo ls -l /etc/sudoers.d
    
    -r--r-----.   1 root root  102 Nov  5 18:49 devops
    
    sudo systemctl stop httpd
    
Testing

    su - devops
    
    sudo systemctl status httpd
    
    sudo systemctl restart httpd
    
    sudo systemctl status httpd
    
    
    [devops@localhost ~]$ sudo cat /etc/shadow

    [sudo] password for devops: 

    Sorry, user devops is not allowed to execute '/bin/cat /etc/shadow' as root on localhost.localdomain.

    
    
Simllar questions:

1. Create a user named backup who can mount and unmount the device /dev/sdb1 on /mnt/backup using sudo, but cannot mount or unmount anything else.
    
    backup ALL=(ALL) NOPASSWD: /usr/bin/mount /dev/sdb1 /mnt/backup, /usr/bin/umount /mnt/backup

2. Create a user named operator who can restart and check the status of the SSH service using sudo.

    operator ALL=(ALL) NOPASSWD: /bin/systemctl restart sshd.service, /bin/systemctl status sshd.service

3. Create a user named ops who can execute the backup script /usr/local/bin/daily_backup.sh with sudo privileges and nothing else.

    ops ALL=(ALL) NOPASSWD: /usr/local/bin/daily_backup.sh
    
    

''''' system users (service accounts) ''''''


**What Are System Users?**

System users are **non-login users** created to **own or run services, scripts, or background processes**, not for interactive login.
They typically:

* Have **no home directory**
* Use **`/sbin/nologin`** or **`/bin/false`** as their shell
* Have **UIDs below 1000**
* Are used by system services like `apache`, `nginx`, `sshd`, etc.

You can see them in `/etc/passwd`:

```bash
grep -E 'nologin|false' /etc/passwd
```

---

**Creating a System User**

To create one manually:

```bash
sudo useradd -r -s /sbin/nologin -M scriptuser
```

**Flags:**

* `-r` → system account
* `-s /sbin/nologin` → prevent login
* `-M` → no home directory

---

**Why Use System Users for Scripts?**

System users are used when you want:

1. **A secure, isolated identity** to run a service or scheduled job (not a real person).
2. **Controlled sudo permissions** (can run one specific script only).
3. **Auditing** — any script action is logged as that user, not `root`.

This is ideal for:

* Automated backups
* Monitoring scripts
* CI/CD pipeline jobs
* Background daemons

---

**Example 1 — System User to Run a Script**

**Task**

Create a **system user** `backupsvc` that can run a backup script `/usr/local/bin/daily_backup.sh` using sudo — but nothing else.

✅ Steps

```bash
sudo useradd -r -s /sbin/nologin -M backupsvc
sudo touch /usr/local/bin/daily_backup.sh
sudo chmod 700 /usr/local/bin/daily_backup.sh
sudo chown root:root /usr/local/bin/daily_backup.sh
```

Then create a sudoers file:

```bash
sudo visudo -f /etc/sudoers.d/backupsvc
```

**Add this line:**

```bash
backupsvc ALL=(ALL) NOPASSWD: /usr/local/bin/daily_backup.sh
```

Now, if a script or cron job runs as `backupsvc`, it can execute:

```bash
sudo /usr/local/bin/daily_backup.sh
```

But it cannot execute **any other** privileged command.

---

**Example 2 — System User to Manage a Specific Service**

**Task**

Create a **system user** `websvc` that can restart only `httpd`.

✅ Steps

```bash
sudo useradd -r -s /sbin/nologin -M websvc
sudo visudo -f /etc/sudoers.d/websvc
```

**Add:**

```bash
websvc ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd, /bin/systemctl status httpd
# websvc ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd.service, /bin/systemctl status httpd.service
```

Now you can safely run:

```bash
sudo -u websvc sudo systemctl restart httpd
```

This allows automation tools (like Ansible, Jenkins, or cron) to restart the service **without giving full root access**.

---

**Example 3 — Running Script Automatically via systemd**

You can pair system users with **systemd service units**.

### Example

Create `/etc/systemd/system/daily-backup.service`:

```ini
[Unit]
Description=Daily Backup Script

[Service]
Type=oneshot
User=backupsvc
ExecStart=/usr/local/bin/daily_backup.sh
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl start daily-backup.service
```

✅ This runs your script **as the `backupsvc` user**, isolated from root.

---

**Summary**

| Feature                          | Normal User            | System User                    |
| -------------------------------- | ---------------------- | ------------------------------ |
| Purpose                          | Human login            | Service/script identity        |
| Has home dir                     | Yes                    | No                             |
| Login shell                      | /bin/bash              | /sbin/nologin                  |
| UID range                        | ≥ 1000                 | < 1000                         |
| Typical use                      | Admin work, sudo tasks | Daemons, cron jobs, automation |
| Can use in sudoers               | ✅ Yes                 | ✅ Yes                         |
| Used for RHCSA privilege control | ✅ Yes                 | ✅ Yes                         |

---



### Working with text files

create file using touch and redirection
    
    touch file1
    
    echo hello > file2
    
    Multiple files:
    
        touch file{1..5}
        
        rm -rf file{1..5}
    
    
Shell output is normally sent to the console for both STDOUT and STDERR.	
Understand STDIN (0), STDOUT (1), STDERR (2) and how to redirect them:

> — redirect STDOUT (overwrite)

>> — append STDOUT

2> — redirect STDERR

&> — redirect both STDOUT and STDERR

Examples:

    ls -la /etc/hosts > output_file1
    
    ls -la /etc/Hosts 2> error_file1
    
    ls -la /etc/hosts /etc/Hosts &> stdout_stderr_file
    
    ls /root > success.txt 2> error.txt
    
    
(Heredoc) is a type of redirection that allows you to pass multiple lines of input to a command — great for scripting.

Example:

````bash
cat > file1 << EOF
Line 1
Line 2
EOF
````

Note:
<< EOF means: read input until the word EOF appears again.


Redirecting with command "tee"

tee is extremely useful when you need to write to files as root via sudo because:

sudo echo "text" >> /etc/file.txt

fails — the >> redirection happens before sudo, so it’s still run as the unprivileged user.

Instead:

echo "text" | sudo tee -a /etc/file.txt


✅ works because tee runs with sudo privileges.

Options:

tee → overwrite

tee -a → append


example:

````bash
cat << EOF | sudo tee /etc/motd
Welcome to the server!
Authorized access only.
EOF
````

note: This uses both heredoc and tee for a privileged write.


**Reading text files**
----------------------

cat

head

tail

less

grep

wc -l /file1  (numner of lines in the file)


Question:
Redirect both STDOUT and STDERR of /usr/bin/find /etc -name passwd into /tmp/find_output.log


touch /tmp/find_output.log

find /etc -name passwd &>> /tmp/find_output.log

or

find /etc -name passwd >> /tmp/find_output.log 2>> /tmp/find_output.log # the difference in order of information

cat /tmp/find_output.log


---


**Text Editors**
----------------
nano
vim

note: vimtutor


sudo yum install -y vim-enhanced

we can add abbrevations into ~/.vimrc  # shortcuts

ex:
    echo 'abbr _sh #!/bin/bash' >> ~/.vimrc

---


### Working with Directories and Files

How to know current working directory?

    pwd

How to create Directories and files?
    
    mkdir
    
    mkdir -p (p for parent)

    Example:
    
    mkdir -p /home/lab/dir{1..3}
    
    touch /home/lab/dir{1..3}/file{A..C}.txt
    

How to delete Directories and files?

    rmdir (used to delete empty directory)

    rm -r (recursively delete directories and files inside it)
    
    rm -i /home/lab/dir1/fileA.txt 
    

How to change the directory?

    cd .. (one directory back)
    
    cd ../.. (two directories back)
    
    cd (user home directory) or cd ~
    
    cd - (previous directory)



**File globbing:** file*, file?, {1..5}

    ls /home/lab/dir*/file?.txt
    

**File operations**

copy files

    cp <source> <destination>  (read permissions for source directory, write permissions to destination directory)

rename or move files

    mv <source> <destination>	(write permissions for both source and destination directory)

delete files

    rm (write permissions for directory)

    rm -i (i for interactive way deleting files)
    

Question:
    Create a directory /archive and move all .log files from /var/log that are older than 7 days.


---



### Securing files in the Filesystem

**What is the Filesytem in Linux?**

The Linux file system is a method of storing and organizing data in a single, unified hierarchical tree structure starting from a single root directory (/), where everything, including hardware devices and processes, is treated as a file. 

It is a logical organization that is independent of the specific **physical file system types (like ext4 or XFS)** used on the underlying storage devices.

In simple words:

A file system is a method that the operating system uses to:

- **Store and organize data** on storage devices (like HDDs, SSDs, USB drives).
- **Manage how data** is read, written, and accessed.

Note:

I found the below links very useful to deepen the understanding of Filesystem:

- https://dev.to/prodevopsguytech/understanding-the-linux-filesystem-an-in-depth-guide-for-devops-engineers-ona
- About File system hierarchy standard: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html


**File types**

    - -> regular file

    d -> directory

    l -> link (Symbolic link)

    p -> pipes (Used for inter-process communication (e.g., /dev/fd/).)

    b -> Block Devices: Represent devices that are accessed randomly, like hard drives (e.g., /dev/sda)

    c -> Character Devices: Represent devices that are accessed sequentially, like keyboards and mice (e.g., /dev/tty)

    s -> Sockets: Used for network communication (e.g., /dev/log)


**About File permissions in Linux**

    read - 4 (decimal), 100 (binary) - read a file or list directory content

    write - 2 (decimal), 010 (binary) - create or delete files in directories, write to existing file 

    execute - 1 (decimal), 001 (binary) - enter a directory or execute program or script


default file permissions - 0666

default directory permissions - 0777

umask -> based the user umask value the default permissions change

Example scenarios:

umask

if umask value of user 0002:

    default file permissions of user - 0664

    default directory permissions of user - 0775
    
if umask value of user 0022:

    file permissions - 0644

    directory permissions - 0755
    

How to set umask value?

    umask 000

    umask 077 

    ...


**Listing the file permissions**

    ls		-> (list without metadata)
    ls -l	-> (List with metadata and no hiden fiels info, l for long)
    ls -la	-> (List with metadata and hiden files, a for all)
    ls -ld	-> (only directories list, d for directory)


    stat - display file or file system status 
    -----------------------------------------

        stat /etc
        
        stat -c %a /etc (-c configuration values, %a numeric version of permissions)
        
        stat -c %A /etc (%A sysmbolic format)


#### Set and Changing file permissions

**Changing/set the file permissions - chmod**

    chmod -v (-v display the old and new permissions in the terminal)

    Example 01: Change the file permissions to a paticular file

        touch file 1

        chmod -v 666 file1


    Example 02: First set permissions using umask, before Creating the files or directories

        umask 007 -> means default file permission 660, default directory permission 770

        mkdir -p upper/{dir1, dir2} or mkdir -p upper/dir{1..2}

        touch upper/{dir1, dir2}/file1 # Creating file1 in both directories (dir1, dir2)

        ls -lR upper  (R - recursively)

        ````bash
        [user1@localhost ~]$ ls -lR upper
        upper:
        total 0
        drwxrwx---. 2 user1 user1 19 Nov  7 09:51 dir1   (770)
        drwxrwx---. 2 user1 user1 19 Nov  7 09:51 dir2   (770)

        upper/dir1:
        total 0
        -rw-rw----. 1 user1 user1 0 Nov  7 09:51 file1	(660)

        upper/dir2:
        total 0
        -rw-rw----. 1 user1 user1 0 Nov  7 09:51 file1	(660)
        ````

        check the difference:

        ````bash
        chmod -vR +x upper
        
        [user1@localhost ~]$ chmod -vR +x upper
        mode of 'upper' retained as 0770 (rwxrwx---)
        mode of 'upper/dir1' retained as 0770 (rwxrwx---)
        mode of 'upper/dir1/file1' changed from 0660 (rw-rw----) to 0770 (rwxrwx---)
        mode of 'upper/dir2' retained as 0770 (rwxrwx---)
        mode of 'upper/dir2/file1' changed from 0660 (rw-rw----) to 0770 (rwxrwx---)
        ````

        ````bash
        chmod -vR a+x upper    (a for all objects, x execute permision)

        [user1@localhost ~]$ chmod -vR a+X upper 
        mode of 'upper' changed from 0770 (rwxrwx---) to 0771 (rwxrwx--x)
        mode of 'upper/dir1' changed from 0770 (rwxrwx---) to 0771 (rwxrwx--x)
        mode of 'upper/dir1/file1' changed from 0770 (rwxrwx---) to 0771 (rwxrwx--x)
        mode of 'upper/dir2' changed from 0770 (rwxrwx---) to 0771 (rwxrwx--x)
        mode of 'upper/dir2/file1' changed from 0770 (rwxrwx---) to 0771 (rwxrwx--x)
        ````

**Manage File Ownership - chown, chgrp**

chown - change both user and group ownership

chgrp - change group ownership

id - print real and effective user and group IDs

Example:

    touch file1

    sudo chown user:group file1

    sudo chgrp group file1


**Link Files**

2 types of link files are available in the Linux.
1. Hard link files
2. Soft link files

Hard links are just extra names linked to the same metadata that means **Another name pointing to the same inode**.

Soft links are a special file type that links to the destination file (A separate file that points to the path of another file). This is a completely a new file that is used as a link to the target. The file type shows as "l" for link.


mkdir -p upper/{dir1, dir2}

ls -ldi upper upper/.    # (i - Inode, d - directories)

ls -ldi upper upper/. upper/dir1/.. upper/dir2/..

upper and upper/. Inode is same (Hard link)



Soft Link (Symbolic Link)

example:

    ln -s /etc/services --> this creates a new file "services" in the current directory

    ls -l services --> this is a link file, that links to /etc/services 

    or

    ln -s /etc/services ports (ports is destination)



**Soft Link (Symbolic Link) vs Hard Links**

Question:

    ln testfile hardlink1

    ln -s testfile softlink1

    Delete the original file and observe behavior difference.

Answer:

````bash
    touch testfile
    echo "This file used for practice links." > testfile
    chmod 640 testfile
    chown user1:devops testfile

    ln testfile hardlink1
    ln -s testfile softlink1

    ls -li testfile hardlink1 softlink1

        123456 -rw-r----- 2 user1 devops  35 Nov  5 21:40 hardlink1
        123456 -rw-r----- 2 user1 devops  35 Nov  5 21:40 testfile
        123789 lrwxrwxrwx 1 user1 user1    8 Nov  5 21:40 softlink1 -> testfile
````

    👉 Notice:

    testfile and hardlink1 have the same inode (123456) and link count = 2

    softlink1 has a different inode (123789) and just stores a pointer -> testfile


    When you delete testfile:
````bash
    rm testfile
````

    - The directory entry testfile is removed.

    - The inode is deleted only if no other hard link references it.

    Since **hardlink1** still references that inode:

    - The file still exists (data intact!)

    - You can still cat hardlink1 ✅

    But **softlink1** points to the name testfile, which no longer exists:

    - The link becomes broken

    - It turns red or flashing (depending on your terminal theme)

    Access fails:

        cat softlink1

        cat: softlink1: No such file or directory

Question:

Create a soft link **/tmp/passlink to /etc/passwd** and a hard link **/tmp/shadowlink to /etc/shadow**. Explain which one succeeds and why.

answer:

````bash
[user1@localhost ~]$ ln -s /etc/passwd /tmp/passlink

[user1@localhost ~]$ ln /etc/shadow /tmp/shadowlink
ln: failed to create hard link '/tmp/shadowlink' => '/etc/shadow': Operation not permitted

[user1@localhost ~]$ ls -l /tmp
total 0
lrwxrwxrwx. 1 user1 user1 11 Nov  7 11:10 passlink -> /etc/passwd
drwx------. 3 root  root  17 Nov  7 08:41 systemd-private-22d22baaae2b441493479c3ac9ef7643-chronyd.service-SsmSsV
drwx------. 2 user1 user1  6 Oct 29 18:32 Temp-33c97bb4-2e26-48be-aa1d-738f7fbb138d
drwx------. 2 root  root   6 Nov  1 11:52 vmware-root_910-2697139510


[user1@localhost ~]$ cat /tmp/passlink 
````

Troubleshooting:

````bash
[user1@localhost ~]$ ls -l /etc | grep shadow
----------.  1 root root      1360 Nov  5 18:38 shadow
----------.  1 root root      1338 Nov  5 18:34 shadow-

[user1@localhost ~]$ ls -l /etc | grep passwd
-rw-r--r--.  1 root root      2030 Nov  5 18:38 passwd
-rw-r--r--.  1 root root      1977 Nov  5 18:34 passwd-
````

Notice: 

The shadow file doesn't have any permissions for others, where as passwd as read permissions to others. so used **sudo**

````bash
[user1@localhost ~]$ sudo ln /etc/shadow /tmp/shadowlink
[sudo] password for user1: 

[user1@localhost ~]$ ls -l /tmp
lrwxrwxrwx. 1 user1 user1   11 Nov  7 11:10 passlink -> /etc/passwd
----------. 2 root  root  1360 Nov  5 18:38 shadowlink

[user1@localhost ~]$ ls -l /etc | grep -e "shadow"
----------.  2 root root      1360 Nov  5 18:38 shadow
----------.  1 root root      1338 Nov  5 18:34 shadow-
````


🧱 Hard Link:

You’re literally telling Linux:

“Create another name for this inode.”

So Linux needs:

- To read the original file’s inode → requires execute permission on its directory (to traverse it).

- To add a new entry in the target directory → requires write + execute on the destination directory.

It doesn’t need to “read” the file data, but it must access the inode.


🪶 Soft Link:

You’re telling Linux:

“Create a tiny file containing this path name.”

So Linux only needs:

- Write access to the directory where you’re placing the link (to create the file).

- It doesn’t even check if the target exists or if you can access it!




User and Group Management
-------------------------

````bash
sudo usermod -aG wheel user1 # adding user1 to wheel group

id user1
````

Note: If the group details are not updated, than Logout and Login to see the changes

**Create User and group**

````bash
# creating user with home directory (/home/alice)
sudo useradd -m alice
sudo passwd alice

# creating a new group
sudo groupadd devops
# Add user account to the group
sudo gpasswd -a alice devops

# adding user (alice) to group (devops)
sudo usermod -aG devops alice
````


**Switch groups - sg, newgrp**

touch file1

ls -l file1

newgrp wheel # opens new shell

or

sg wheel

touch file 2

ls -l file2

exit # return to previous shell

Check the file1 and file2 group names


**Switch users**

su  (switched to the root user, your working directory won't change)

The below both commands gives a full login shell and start in the root user's home directory

su - 

su -l 

su - user1


Question:

Add a user developer who has a private group devteam and a home directory /home/devdir. Then add this user to the wheel group.

sudo groupadd devteam

sudo useradd -m -d /home/devdir -g devteam developer

sudo passwd developer

sudo usermod -aG wheel developer

````bash
[user1@localhost ~]$ id -Gn developer
devteam wheel

[user1@localhost ~]$ cat /etc/passwd | grep developer
developer:x:1003:1003::/home/devdir:/bin/bash

[user1@localhost ~]$ groups developer
developer : devteam wheel

[user1@localhost ~]$ id developer
uid=1003(developer) gid=1003(devteam) groups=1003(devteam),10(wheel)
````
---

**write-only files**

````bash
touch log

ls -l log

chmod -v u=w log  (u - user)

ls -l log (it works)

cat log (permision denied, because user have only write permision)

echo "new line added" >> log

sudo cat log 
````


**Securing directories**

mkdir -m 155 project1 (m - mode, 1 for user, 5 for group, 5 for others)

ls -l project1 (permision denied, required r)

ls -ld project1 (Success, because of x)

cd project1 (sucess, becuase of x)

ls (permision denied, required r)

cd ..

chmod -v u=wx project1

echo "new information into new file" > project1/file1 (sucess, because w + x)

cat project1/file1   (this command works, because x on directory level, r on file level)

-rw-rw----. 1 user1 user1 30 Nov  7 15:39 file1

ls project1 (permission denied, because user doesn't have read permision)


Note:

Short, clear rules to memorize:

To list directory contents: r on the directory.

To enter a directory or access files by name: x on the directory.

To create/delete files inside: w + x on the directory.

To read a file: r on the file (and x on the directory).

To delete a file: w on the directory (file’s own perms don’t matter for deletion).



### Archiving Files in Linux

- archiving files using tar and star
- file compression using gzip and bzip2


**tar - Tape Archives**

tar can be used to create file archives. tar file is not compressed but may appear to be a slightly small size than the original content. this is due to the more efficient use of blocks in the filesystem.

````bash
sudo du -sh /etc (du - disk usage, s - size of summary)

sudo tar -cf etc.tar /etc 

ls -lh etc.tar
````

-c for create

-t table of contents

-x extract

-f archive file


**star**

sudo yum install -y star


**File compression**

2 utilites

    gzip / gunzip

    bzip2 / bunzip2
    

tar -czf (z for gzip)

tar -cjf (j for bzip2)

to extract:

tar -xzf etc_backup.tar.gz -C /tmp/restore


Example:

sudo tar -cf etc.tar /etc

ls -lh etc.tar

time gzip etc.tar

ls -lh

to expand the gzip file (unzip)

    gunzip etc.tar.gz
    
Simllary

time bzip2 etc.tar

bunzip2 etc.tar.bz2 # unzip


Questions:

**Create a compressed archive /root/etc_backup.tar.bz2 of /etc excluding /etc/selinux.**

sudo tar -cjf /root/etc_backup.tar.bz2 --exclude=/etc/selinux /etc

**Create a directory /data/private where only the owner can list and access files, but others can’t even see filenames.**

mkdir -p -m 500 /data/private

(list r + access x)


## Module 02 - Bash/Shell Scripting

- Script interpreter/shell interpreter (shebang)


**Run the Loops with CLI commands**
Traditional CLI commands for loops (for, while)

    for i in Hallo pavan Tuschss; do echo $i; done
    
        note: Hallo pavan Tuschuss is list with size 3
    
    i=4 ; while [ $i -gt 0 ]; do echo $i; let i-=i; done
    

How to know the type?

    type for  # this provides the type information about the for

How to know the type of file?

    file <filename.extension>
    


Example:

    Create 3 users with home directories?

    for user in pavan ram hari; do sudo useradd -m $user; echo Password1 | sudo passwd --stdin $user; tail -n1 /etc/passwd; done

    Delete 3 users and also their home directories?

    for user in pavan ram hari; do sudo userdel -r $user; tail -n3 /etc/passwd; done

### working with bash exit codes and simple logic

**Exit Codes**

0 -> is success

variable-> $? -> provides recent command exit code

echo $?

getent passwd bob ; echo $? # if user exit than value 0


**simple logic**

There was explantion behind the "simple logics" used in the loops.

    && -> used to combine two commands, but we need to understand - "The second command only runs if the first command succeeds"
    
    mkdir dir1 && cd dir1
    
    || -> the second command only runs if the first command fails

    cd dir || mkdir dir1

Example:

    cd mkt || mkdir mkt && cd mkt  # you get error message, if cd mkt file doesn't exist. the right part after || creates and cd to mkt

    pwd

    cd 

    rm -r mkt

    The previous command improved

    cd mkt 2>/dev/null || mkdir mkt && cd mkt  # you doesn't get error message because error redirected to /dev/null

    pwd

    cd  # returns to user home directory

    cd mkt 2>/dev/null || mkdir mkt && cd mkt # you got cd error message, becuase each command independent 

    what happening in the background?:

        cd mkt 2>/dev/null || mkdir mkt  # cd mkt success so skip the mkdir mkt, than
    
        cd mkt # error because no mkt directory inside mkt direcctory (mkt/mkt)

    solution: grouping the commands

````bash
    cd 

    rm -r mkt

    [user1@localhost ~]$ cd mkt 2>/dev/null || { mkdir mkt && cd mkt; }
    [user1@localhost mkt]$ pwd
    /home/user1/mkt
    [user1@localhost mkt]$ cd 
    [user1@localhost ~]$ cd mkt 2>/dev/null || { mkdir mkt && cd mkt; }
    [user1@localhost mkt]$ pwd
    /home/user1/mkt
    [user1@localhost mkt]$ cd mkt 2>/dev/null || mkdir mkt && cd mkt
    [user1@localhost mkt]$ pwd
    /home/user1/mkt/mkt
````


**PATH Environment variable**

we can run the script irrespective of file path. To do this we need add the file path to PATH environmental variable.

[user1@localhost ~]$ $PATH

bash: /home/user1/.local/bin:/home/user1/bin:/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin: No such file or directory


Example

    touch my.sh

    mkdir bin && mv my.sh bin/

    chmod -v +x my.sh

    which my.sh

    [user1@localhost ~]$ which my.sh

    ~/bin/my.sh


 
**Special variables**

    $$ - current PID

    $? - exit status of previous command

    !$ - Last argument  (useful in scripts)

    $0 - program names

    $1 - first argument

    $# - arguments count

    $* - all arguments as a string

    $@ - all arguments as an array


- creating scripts and processing arguments
- reading user input during execution
- using logic and looping syntax
- using functions to improve your code

**Scripting - Automating the user creation process**

Question:

Create a script taking an argument for the username, the script should not proceed if the name is not supplied. Use conditional statements to ensure we only try to create the account if it does not already exit. The password will be collected during script execution using the read command. For confirmation, the new user account deatils are printed to the screem.


### Loops and Functions

---


## Module 03 - Operating running systems

### Rebooting and shutting down systems from the CLI

- shutdown 

    * shutdown command
    * restricting user login

    shutdown [OPTIONS...] [TIME] [WALL...]

        Shut down the system.

        Options:
            --help      Show this help
        -H --halt		Halt the machine
        -P --poweroff	Power-off the machine
        -r --reboot		Reboot the machine
        -h				Equivalent to --poweroff, overridden by --halt
        -k				Don't halt/power-off/reboot, just send warnings
        --no-wall		Don't send wall message before halt/power-off/reboot
        -c				Cancel a pending shutdown
        --show			Show pending shutdown

    Examples:

    Sudo shutdowm 20+ "shutting down in 20 minutes"

    sudo shutdown 17:00 "system going down at 5 pm"

    sudo shutdown now "System going down"

- reboot
- poweroff

**Restricting User Access:**

Creating the file **/etc/nologin** standard users are restricted from logging into the system. 

Using the shutdown command, standard users are restricted from login when less than 5 mintues remain before the event. This is controlled via **/run/nologin** (ls /run/nologin).


Example:

````bash
sudo touch /etc/nologin

The above command restrict the users login after the file creation even it is an empty file. The current login is not terminated. This helpful in server maintaince time, not allow users to login into system.

C:\Users\User>ssh user1@192.168.88.128
user1@192.168.88.128's password:
Connection closed by 192.168.88.128 port 22

sudo rm /etc/nologin

C:\Users\User>ssh user1@192.168.88.128
user1@192.168.88.128's password:
Last failed login: Sat Nov  8 19:57:31 CET 2025 from 192.168.88.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Sat Nov  8 19:56:36 2025 from 192.168.88.1
````

````bash
[user1@localhost ~]$ shutdown -r +2 "Rebooting the system"
Reboot scheduled for Sat 2025-11-08 19:46:04 CET, use 'shutdown -c' to cancel.

[user1@localhost ~]$ ls /run/nologin
/run/nologin

[user1@localhost ~]$ sudo cat /run/nologin
System is going down. Unprivileged users are not permitted to log in anymore. For technical details, see pam_nologin(8).
````

**Another way to Reboot / Poweroff the system**

````bash
sudo systemctl poweroff

sudo systemctl reboot

[user1@localhost ~]$ ls -l $(which reboot)
lrwxrwxrwx. 1 root root 16 Jan 28  2025 /usr/sbin/reboot -> ../bin/systemctl

[user1@localhost ~]$ ls -l $(which poweroff)
lrwxrwxrwx. 1 root root 16 Jan 28  2025 /usr/sbin/poweroff -> ../bin/systemctl
````

The Disadvantage:

These are immediate actions, so users won't get any time or message to save their work. These commands are not used in the real-time environments.


### Recovering root password

**Boot Process**

power on system, this starts - BIOS and BIOS locate boot partition - the boot partition should GRUB loded in it, the GRUB boot loader start Linux kernel - the kernel is loaded but prior to this the initialization ram disk is loaded to customize the boot process to your hardware (drives).


**Interrupting the boot process && Resetting the root password**

The scenario of recovering root password:

If a system is not used frequently, it is possible the root password may become forgotten.

Process:

In VMWare workstation / Oracle Virtual box

1. VM → Poweron

2. Press Esc repeatedly (immediately, when the VM window appear), than e (for edit)

3. Add additional kernel boot arguments 

    rd.break -> the argument used to break the boot process, so we can remount /sysroot with read and write permissions and reset the root password


    enforcing=0 -> allowing errors, when the root login into the system initial time. After that set enforcing 1 (in techinical terms: In permissive mode, SELinux logs policy violations but doesn’t enforce them. Always re-enable enforcing mode after fixing issues)

![Example](./imgs/grub_1.png) 

4. remount /sysroot with read, write permissions

![Example](./imgs/grub_2.png) 

5. Reset the boot process and continue the boot process

![Example](./imgs/grub_3.png)

````bash
[root@localhost user1]# getenforce
Permissive
[root@localhost user1]# ls -Z /etc/shadow
system_u:object_r:unlabeled_t:s0 /etc/shadow
[root@localhost user1]# restorecon -v /etc/shadow
Relabeled /etc/shadow from system_u:object_r:unlabeled_t:s0 to system_u:object_r:shadow_t:s0
[root@localhost user1]# setenforce 1
[root@localhost user1]# getenforce
Enforcing
````

**SELinux and file context**

SELinux Contexts protect files by labeling them with a type, user, and role.

If the **/etc/shadow** file label is wrong (e.g., changed from shadow_t to user_home_t), authentication fails because PAM can’t read it.

**restorecon** command resets it to the correct context from SELinux policy.

Note: Do only on the Lab system not in production systems

Switch user as root

sudo -i 

next steps:

sudo cat /etc/shadow # this is the place to store the user passwords in hashed format

ls -Z /etc/shadow # SELinux context

chcon -t user_home_t /etc/shadow # chcon - change context

ls -Z /etc/shadow

sudo -i -u user1  # you will get PAM error, because can't authenticate through to the shadow file

restorecon -v /etc/shadow # v for verbose

sudo -i -u user1 # it will work


````bash
[root@localhost ~]# ls -Z /etc/shadow
system_u:object_r:shadow_t:s0 /etc/shadow
[root@localhost ~]# chcon -t user_home_t /etc/shadow
[root@localhost ~]# ls -Z /etc/shadow
system_u:object_r:user_home_t:s0 /etc/shadow
[root@localhost ~]# sudo -i -u user1
sudo: PAM account management error: Authentication service cannot retrieve authentication info
sudo: a password is required
[root@localhost ~]# restorecon -v /etc/shadow
Relabeled /etc/shadow from system_u:object_r:user_home_t:s0 to system_u:object_r:shadow_t:s0
[root@localhost ~]# sudo -i -u user1
[user1@localhost ~]$ 
````


### Managing system services using systemctl

systemd as the service manager


- start/stop

- enable/disable/--now (enable --now -> enable and start the service, diable --now -> disable and stop the service)

- status

**Unit files**
 
unit file types: service, socket, timer, and target 

Unit file location:

- /usr/lib/systemd/system -> the standard location of unmodified unit file

- /etc/systemd/system -> modified or custom unit files overwrite the defaults when added to this path


List the unit files:

- systemctl list-units

- systemctl list-units --type socket

- systemctl list-units --type target

- systemctl list-unit-files --type socket


Note: 

list-units = loaded systemd units

list-unit-files = All unit files no matter if they have been loaded or not


Read and Edit unit files:

systemctl cat <service name>

ex: systemctl cat sshd

editing:

sudo systemctl edit --full sshd (--full = full copy of original file)

systemctl cat sshd (now The location is /etc/systemd/system)

sudo systemctl daemon-reload


How to do Mask the service:

1. delete customization

sudo systemctl rm /etc/systemd/system/sshd.service

sudo systemctl daemon-reload

systemctl cat sshd (now the location is usr/lib/systemd/system/sshd.service)

2. mask

sudo systemctl mask sshd --now

sudo systemctl start sshd

Note: mask is useful to wantedly not use the service for some time/task without uninstall the service


**Targets replace runlevels used in earlier versions of RHEL**

default run level (booting time): systemctl get-default

Change the default: sudo systemctl set-default graphical.target


### Adjusting system performance

**How busy is the system**

- uptime
- top / htop

To check the no.of CPUs and Cores:

    lscpu | grep -E '^(CPU\(s\)|Core\(s\))'


**Adjusting CPU priority**

- nice (-20 to +19, which adjust the priorty from 60 to 99. 99 is the lowest priority)

- renice
- jobs

The nice values of the processes helpful to change the CPU priority of the processes. There exit a releation between process nice value and CPU priority.

Example:

sleep 1000& -> running the command in background

jobs # to check the jobs

ps -elf | grep sleep

or

pgrep sleep # pgrep especially used to search in the processes

check the priority value of the process:

ps -lp $(pgrep sleep)

pkill sleep


**with nice value**

nice -n 12 sleep 1000&

ps -lp $(pgrep sleep)

renice 19 <PID>


**Managing processes**

- ps, pgrep, pkill, kill

ps

ps -elf (e -> every thing, l -> long lsit)

ps -fp 1 (ps -> process status, f -> full list, p -> process id)

kill -l # list of kill signals

ss -ntlp


**Tuning profiles**

tuned-adm active

tuned-adm recommend

tuned-adm list




### Managing logs

For example your services are not running/not working, what you will do/where you will search? [debuging]

1. look logs from the **systemctl status**
- check service **active, enabled, status, logs**

2. /var/log, /var/log/messages


3. journalctl
- I don't need log file information


Note: 

The tradetional logging mechanism within modern RHEL is **Rocket Fast Syslog Daemon (rsyslogd)**

- less /etc/rsyslog.conf # default configuration file

- grep 'rsyslog.d' /etc/rsyslog.conf 

we can make custom configuration by creating **/etc/rsyslog.d/my.conf**

ex: /etc/rsyslog.d/my.conf

enter:

local0.info /var/log/my.log 

level0 to level7 are the facilities used for different services (local use).

sudo systemctl restart rsyslog.service

ls -l /var/log

logger -p local0.info "test rsyslog" # creates my.log file and log message entry

ls -l /var/log

sudo tail /var/log/my.log

For more information: 

man 3 syslog # documentation

man 5 rsyslog.conf


**Rotate log file using logrotate**

The command **logrotate** is used to maintain the size of the **/var/log** structure. 

It is run by cron daemon or manually. The default configuration file /etc/logrotate.conf, for custom configuration /etc/logrotate.d/<your_file_name.conf>

/var/log/my.log
{
    weekly 
    rotate 4
    size 100M
    dateext
    compress
    copytruncate
}

sudo logrotate /etc/logrotate.conf 

or

sudo logrotate /etc/logrotate.d/my.conf

ls -l /var/log/my*


man 5 logrotate.conf


**journalctl**

sudo journalctl

sudo journalctl -n5

sudo journalctl --since yesterday

sudo journalctl --since yesterday --unit sshd


Note:

The journal logs are stored in memory and may not persist as disk files.

How to make persistance?

grep 'Storage' /etc/systemd/journald.conf

sudo sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf

grep 'Storage' /etc/systemd/journald.conf

sudo systemctl restart systemd-journald


sudo journalctl -b -1 (b -> boot process, -1 previous boot details)

**copy logs other files securiy with scp (linux)or winscp (windows)**


try to copy the files using 
- user and password authentication
- user and ssh key public key authentication


## Module 04 - Configuring Local Storage

### Block Storage

Block storage refers to **storage devices** that read/write data in fixed-size blocks (usually 512 bytes or 4 KB).

Examples of block storage devices:

- Hard disks (HDD)

- SSDs

- NVMe drives

- USB drives

- LVM volumes

- Loop devices (virtual block device backed by file)

- SAN/iSCSI disks


Location of Block and character devices (/dev)

Block devices (b) → under /dev (e.g., /dev/sda, /dev/nvme0n1, /dev/loop0)

Character devices (c) → under /dev (e.g., /dev/tty, /dev/null) # handles 1 character or byte at a time

**Block Device Architecture**

Physical Device → Device Driver → Device File (/dev/sda represents the disk)

```sql
+------------------+
|   Physical Disk   |   (HDD/SSD/NVMe)
+------------------+
           |
           v
+------------------+
|   Device Driver   |  (kernel module: sd_mod, nvme, loop, etc.)
+------------------+
           |
           v
+------------------+
| Device File /dev/sda |
+------------------+
           |
           |
     User Applications
```

The device driver is the kernel-side translator between hardware & Linux processes.


Data flow: 

Ex 1: 

cat file1.txt (reading from disk to terminal)

```
Application (cat)
       |
       v
Virtual Filesystem Layer (VFS)
       |
       v
Filesystem driver (ext4/xfs)
       |
       v
Block layer
       |
       v
Device Driver (sd_mod, nvme, loop, etc.)
       |
       v
Hardware (Disk)
```

Then the data goes back up the stack:

Disk → driver → block layer → fs → VFS → cat → stdout → terminal driver → screen


For more information: https://opensource.com/article/16/11/managing-devices-linux 

lsmod -> list of loaded kernel modules (drives + other kernel components)


modinfo sd_mod # Shows metadata for the SATA/SCSI disk driver

modinfo loop # loop device driver info

modinfo nvme # nvme device driver info

lsblk


**Loop Devices**

A loop device lets a file behave like a disk.

List current available loop devices:

losetup -a

Create loop device:

sudo losetup -f <disk> --show

Check list of all block devices:

lsblk

man 4 loop


Windows vs Linux
----------------

Windows:

Application → Windows API → NTFS driver → Storage Stack → Disk Driver → Disk


Linux:

Application → VFS → Filesystem Driver (ext4/xfs/btrfs) → Block Layer → Device Driver → Disk

**Windows Storage Model**

Disk → Partition → Drive Letter

Example:

* Disk 0 = 1TB
* Partitions:

  * C:\ (Windows OS)
  * D:\
  * E:\


**How Windows maps partitions**

Windows uses the **Volume Manager** to assign drive letters.


**How Windows reads/writes data**

```
APP (Notepad, cmd, PowerShell)
   ↓
Windows API (CreateFile, ReadFile, WriteFile)
   ↓
NTFS / exFAT / ReFS filesystem driver
   ↓
Windows Storage Stack
   ↓
Disk Driver (StorAHCI, NVMe, USBSTOR)
   ↓
Physical Disk
```

The user **does NOT interact with hardware directly** — Windows API & filesystem drivers do the work.

✔ Windows hides the device files (Windows does NOT have device files like Linux; it uses "Device Objects" internally).


**Linux Storage Model**

Disk → Partitions → Mounted Directories

Example:

```
/dev/nvme0n1     ---> entire disk  
/dev/nvme0n1p1   ---> /boot
/dev/nvme0n1p2   ---> LVM PV
```

**Linux Data Flow**

```
APP (cat, vim, cp)
   ↓
VFS (Virtual File System layer)
   ↓
Filesystem driver (xfs/ext4)
   ↓
Block Layer (I/O scheduling, merging)
   ↓
Device Driver (kernel module: sd_mod, nvme)
   ↓
Physical Disk
```

Linux uses:

* **VFS** (a generic interface)
* **Filesystem driver**
* **Block I/O layer**

The device file in `/dev` is *not* the entry point to the user. The **filesystem is the entry point**.

Example:

```
cat /home/user/file.txt
```

does *not* go through `/dev/nvme0n1` directly.

The OS maps paths to inodes → filesystem driver → block layer → driver → disk.


**Important**

*The device file is NOT for normal file access*

You do **not** read/write `/dev/nvme0n1p1` when accessing normal files.

Device files are only for:

* Disk utilities (fdisk, mkfs, dd)
* Mounting filesystems
* Low-level operations


**Deep Comparison Table (Windows vs Linux)**

| Concept                               | Windows                   | Linux                          |
| ------------------------------------- | ------------------------- | ------------------------------ |
| How apps access files                 | Windows API               | VFS                            |
| Filesystem drivers                    | NTFS, ReFS, exFAT         | ext4, xfs, btrfs               |
| Hardware access                       | Windows Storage Stack     | Block I/O layer                |
| Disk driver                           | StorAHCI, NVMe            | sd_mod, nvme                   |
| Representation of hardware            | Hidden device objects     | Device files in /dev           |
| Mounting                              | Automatic (drive letters) | Manual mount points            |
| How user sees storage                 | Drives: C:\ D:\ E:\       | Directories: / /boot /home     |
| Can the user read from disk directly? | No                        | Yes, using device files (/dev) |


````mathematica

               WINDOWS FILE ACCESS PATH
 ┌───────────────────────────────────────────────────────┐
 │                 Application (Notepad, cmd)            │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │              Windows API (ReadFile, WriteFile)        │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │         NTFS / ReFS / exFAT Filesystem Driver         │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │                  Windows Storage Stack                │
 │      (Volume Manager, Cache Manager, I/O Manager)     │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │          Disk Driver (StorAHCI, NVMe, USBSTOR)        │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │                      Physical Disk                     │
 └───────────────────────────────────────────────────────┘




                     LINUX FILE ACCESS PATH
 ┌───────────────────────────────────────────────────────┐
 │    Application (cat, vim, cp, rsync, chromium)        │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │        VFS — Virtual File System Layer (generic)      │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │   Filesystem Driver (ext4, xfs, btrfs, vfat, iso9660) │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │  Block Layer (I/O scheduler: mq-deadline, bfq etc.)    │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │     Device Driver (sd_mod, nvme, usb-storage etc.)    │
 └───────────────────────────────────────────────────────┘
                          │
                          ▼
 ┌───────────────────────────────────────────────────────┐
 │                     Physical Disk                      │
 └───────────────────────────────────────────────────────┘
````

---

### Creating and partitioning Block Devices

#### Adding another disk to system

power off the VM in VMWare workstation -> go to VM settings -> select Hard Disk -> click **Add** -> Select **disk type** and **virtual or physical disk** -> ok

check:

power on VM -> lsblk


**Creating Raw Disk Files and loop devices setup**

There an option to create raw disk files using **dd** or **fallocate**

Check which method is more efficient:

time dd if=/dev/zero of=dd.disk bs=1M count=500

if -> input file

of -> output file

bs -> block size

count -> 500 * 1MiB


time fallocate -l 500M fa.disk


sudo losetup -f <disk file> --show # Attach next available loop device

sudo losetup /dev/loop1 <disk file> # Attach loop1

losetup -a # List loop devices

sudo losetup -d /dev/loop1 # Delete or detach loop1

sudo losetup -D # Detach all

sudo rm <disk file>

````bash
time dd if=/dev/zero of=dd.disk bs=1M count=500

time fallocate -l 500M fa.disk

[root@localhost user1]# sudo losetup -f dd.disk --show
/dev/loop0
[root@localhost user1]# sudo losetup -f fa.disk --show
/dev/loop1
[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0           7:0    0   500M  0 loop 
loop1           7:1    0   500M  0 loop 
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
````


#### Disk Partition

To partition disk, first you should know about the **Patition tables**.

- Partition table types

    * Traditional one: MBR or MSDOS Table (Master Boot Record) - Max 2TB size - max 4 primary or 3 primary, 1 extended plus logical
    
    Note: SCSI driver allows Max 15 partitions

    * New one: GPT / GUID partition table - Part of the UEFI framework - max 8ZB size - max 255 partitions per disk

    Note: SCSI driver allows Max 15 partitions

Commands used for disk partition:

fdisk (MBR) / parted

gdisk (GPT)


````bash

# Create partition

root@localhost user1]# 
[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 

[root@localhost user1]# ls -l /dev | grep nvme0n2
brw-rw----. 1 root  disk    259,   4 Nov 14 15:49 nvme0n2

[root@localhost user1]# ls -l /dev | grep nvme0n1
brw-rw----. 1 root  disk    259,   0 Nov 14 15:49 nvme0n1
brw-rw----. 1 root  disk    259,   1 Nov 14 15:49 nvme0n1p1
brw-rw----. 1 root  disk    259,   2 Nov 14 15:49 nvme0n1p2
brw-rw----. 1 root  disk    259,   3 Nov 14 15:49 nvme0n1p3

[root@localhost user1]# sudo fdisk --help

Usage:
 fdisk [options] <disk>         change partition table
 fdisk [options] -l [<disk>...] list partition table(s)

Display or manipulate a disk partition table.

Options:
 -b, --sector-size <size>      physical and logical sector size
 -B, --protect-boot            don't erase bootbits when creating a new label
 -c, --compatibility[=<mode>]  mode is 'dos' or 'nondos' (default)
 -L, --color[=<when>]          colorize output (auto, always or never)
                                 colors are enabled by default
 -l, --list                    display partitions and exit
 -x, --list-details            like --list but with more details
 -n, --noauto-pt               don't create default partition table on empty devices
 -o, --output <list>           output columns
 -t, --type <type>             recognize specified partition table type only
 -u, --units[=<unit>]          display units: 'cylinders' or 'sectors' (default)
 -s, --getsz                   display device size in 512-byte sectors [DEPRECATED]
     --bytes                   print SIZE in bytes rather than in human readable format
     --lock[=<mode>]           use exclusive device lock (yes, no or nonblock)
 -w, --wipe <mode>             wipe signatures (auto, always or never)
 -W, --wipe-partitions <mode>  wipe signatures from new partitions (auto, always or never)

 -C, --cylinders <number>      specify the number of cylinders
 -H, --heads <number>          specify the number of heads
 -S, --sectors <number>        specify the number of sectors per track

 -h, --help                    display this help
 -V, --version                 display version

Available output columns:
 gpt: Device Start End Sectors Size Type Type-UUID Attrs Name UUID
 dos: Device Start End Sectors Cylinders Size Type Id Attrs Boot End-C/H/S Start-C/H/S
 bsd: Slice Start End Sectors Cylinders Size Type Bsize Cpg Fsize
 sgi: Device Start End Sectors Cylinders Size Type Id Attrs
 sun: Device Start End Sectors Cylinders Size Type Id Flags

For more details see fdisk(8).

[root@localhost user1]# sudo fdisk /dev/nvme0n2

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xf7471cdd. 

Command (m for help): m

Help:

  DOS (MBR)
   a   toggle a bootable flag
   b   edit nested BSD disklabel
   c   toggle the dos compatibility flag

  Generic
   d   delete a partition
   F   list free unpartitioned space
   l   list known partition types
   n   add a new partition
   p   print the partition table
   t   change a partition type
   v   verify the partition table
   i   print information about a partition

  Misc
   m   print this menu
   u   change display/entry units
   x   extra functionality (experts only)

  Script
   I   load disk layout from sfdisk script file
   O   dump disk layout to sfdisk script file

  Save & Exit
   w   write table to disk and exit
   q   quit without saving changes

  Create a new label
   g   create a new empty GPT partition table
   G   create a new empty SGI (IRIX) partition table
   o   create a new empty DOS partition table
   s   create a new empty Sun partition table



Command (m for help): n

Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)

Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-20971519, default 2048): 

Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519): +1G

Created a new partition 1 of type 'Linux' and of size 1 GiB.

Command (m for help): p
Disk /dev/nvme0n2: 10 GiB, 10737418240 bytes, 20971520 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xf7471cdd

Device         Boot Start     End Sectors Size Id Type
/dev/nvme0n2p1       2048 2099199 2097152   1G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.


[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:6    0     1G  0 part 


# Delete the partition

root@localhost user1]# sudo fdisk /dev/nvme0n2

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 

````
Steps overview:

* Created a new DOS disklabel 

* Create Partition table
    - Partition type
    - Size
    - Sector details

````bash
sudo parted /dev/nvme0n2 mklabel msdos mkpart primary 0% 25%
````


````bash
[root@localhost user1]# sudo parted /dev/nvme0n2 mklabel msdos
Warning: The existing disk label on /dev/nvme0n2 will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? Yes                                                               
Information: You may need to update /etc/fstab.

[root@localhost user1]# sudo parted /dev/nvme0n2 mkpart primary 0% 25%
Information: You may need to update /etc/fstab.

[root@localhost user1]# lsblk                                             
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part 
````

#### Working with Filesystems

Above the disk is partitioned, to use the partition - we need to add a filesystem.

The problem is **Device Names are Transitory (Device naming issue)**

Transitory means, The disk nvme0n1 may be the nvme0n1 today but if it is not the first disk detected on the next boot it will not be nvme0n1, it could be nvme0n2

Solution:

- Filesystems can be optional assigned with a label to identify them

- All filesystems have a UUID (Universaly Unique ID) that uniquely identifies that filesystem.


Demo:

Adding a filesystem, we can mount the partition or entire disk using persistent names


Make filesystem:

sudo mkfs.xfs -L "DATA" /dev/nvme0n2p1

L -> Label (optional)

sudo mount LABEL=DATA /mnt # mounted using the filesystem label

or

sudo mount PARTLABEL=<partlabel> /mnt


How to check the Disk partition label?

````bash
lsblk -o name,mountpoint,label,size,uuid

sudo fdisk -l
````

mount -t xfs


````bash
[root@localhost user1]# sudo mkfs.xfs -L "DATA" /dev/nvme0n2p1
meta-data=/dev/nvme0n2p1         isize=512    agcount=4, agsize=163776 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=655104, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# Mounting using Filesystem LABEL
[root@localhost user1]# sudo mount LABEL=DATA /mnt
[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part /mnt

# Unmount
[root@localhost user1]# sudo umount /mnt
[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part 

# Mounting using Filesystem UUID

[root@localhost user1]# sudo blkid /dev/nvme0n2p1
/dev/nvme0n2p1: LABEL="DATA" UUID="58187d3a-aaa8-4c34-844d-1d3ef06d446b" TYPE="xfs" PARTUUID="24bd0e25-01"

[root@localhost user1]# sudo mount UUID="58187d3a-aaa8-4c34-844d-1d3ef06d446b" /mnt

[root@localhost user1]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part /mnt

# Make Persistent

sudo mkdir /data

sudo vim /etc/fstab # fstab -> filesystem tables

UUID=58187d3a-aaa8-4c34-844d-1d3ef06d446b /data xfs     defaults        0 0

# if the filesystem type ext4 than 0 1

root@localhost /]# sudo mount -a
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
[root@localhost /]# systemctl daemon-reload
[root@localhost /]# sudo mount -a

[root@localhost /]# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part /data

````

### Creating Dynamic Disks using LVM2

- Persisting Loop Devices
- Understanding LVM
- Configuring the LVM system
- Configuring storage layers (LVM)
- Managing volume groups and volumes



#### Persisting Loop Devices

We already touched the Loop Devices topic. These are additional disks setup from the files, ISO images ...

One problem:

These loop devices are not persistant, when we reboot the system the loop device is not exist anymore. Once again need use **losetup** command. But the file or ISO image... exist inside the system, just need to link the loop device to file or ISO.

Solution:

1. Manual approach using CLI command after reboot the system
    
    - sudo losetup # for linking
    - sudo partprobe # read partition tables from the loop device and load into the memory

2. Automating using systemd unit file, [automatically execute during boot process, so it available]

losetup.service

````
    [Unit]
    Description=Setup loop device
    DefaultDependencies=no
    Before=local-fs.target
    After=systemd-udevd.service
    Required=systemd-udevd.service

    [Service]
    Type=oneshot
    ExecStart=/sbin/losetup /dev/loop1 /var/disks/sidk1
    ExecStart=/sbin/partprobe /dev/loop1
    TimeourSec=60
    RemainAfterExit=no

    [Install]
    WantedBy=local-fs.target
````

sudo systemctl daemon-reload

sudo systemctl enable losetup

sudo reboot

lsblk # check the loop device exist or not


#### About LVM, Storage layers

**Logical Volume Management:**

Aggregating block storage to re-allocate as required/needed in the form of (Logical volumes) device-mapper volumes.

List LVM:

lsblk

List Device-mapper devices:

````bash
sudo dmsetup ls --tree
````

**LVM2 Storage Layers**

3 Layers

- Logical volumes: Dev-mapper devices which are formatted and presented to the consumer as a block device

- Volume groups: Volume groups acts as storage pool, aggregating storage together and overcoming the limitations of physical storage size

- Physical volumes: Physical storage existing on the host as disks, partitions, and raw files


**Managing LVM**

Physical volumes - pvs, pvremove, pvcreate

Volume groups - vgcreate, vgs, vgdisplay

Logical volumes - lvs, lvcreate, lvresize



#### Configuring the LVM system, storage layers (LVM), and Managing volume groups and volumes

I have second disk in my VM

````
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part /data
````
nvme0n2p1 is the 1st partition and created XFS filesystem and added the Filesystem UUID in /etc/fstab for mount persistant.


````
sudo parted /dev/nvme0n2 mklabel msdos

sudo parted /dev/nvme0n2 mkpart primary 0% 25%

sudo blkid /dev/nvme0n2p1

sudo mkdir /data

sudo vim /etc/fstab

UUID=58187d3a-aaa8-4c34-844d-1d3ef06d446b /data xfs     defaults        0 0

# if the filesystem type ext4 than 0 1

sudo mount -a

systemctl daemon-reload

lsblk
````

Now, Creating **Partitions** by specifing type as LVM **Mark partitions as LVM type**:

````bash
sudo parted /dev/nvme0n2 mkpart primary 25% 50% set 2 lvm on

sudo parted /dev/nvme0n2 mkpart primary 50% 75% set 3 lvm on
````

````bash

# The current status

[user1@localhost var]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
└─nvme0n2p1   259:5    0   2.5G  0 part /data

[user1@localhost var]$ sudo parted /dev/nvme0n2 print
[sudo] password for user1: 
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n2: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  2684MB  2683MB  primary  xfs


# New partitions

[user1@localhost ~]$ sudo parted /dev/nvme0n2 mkpart primary 25% 50% set 2 lvm on
Information: You may need to update /etc/fstab.

[user1@localhost ~]$ sudo parted /dev/nvme0n2 mkpart primary 50% 75% set 3 lvm on
Information: You may need to update /etc/fstab.

[user1@localhost ~]$ sudo parted /dev/nvme0n2 print                       
Model: VMware Virtual NVMe Disk (nvme)
Disk /dev/nvme0n2: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  2684MB  2683MB  primary  xfs
 2      2684MB  5369MB  2684MB  primary               lvm
 3      5369MB  8053MB  2684MB  primary               lvm

[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
└─nvme0n2p3   259:7    0   2.5G  0 part 
````

**Creating LVM System, Volumes**

````bash
[user1@localhost ~]$ sudo pvs
  PV             VG   Fmt  Attr PSize  PFree
  /dev/nvme0n1p3 rhel lvm2 a--  58.41g    0 

[user1@localhost ~]$ sudo vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  rhel   1   3   0 wz--n- 58.41g    0 

[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g      

sudo vgcreate vg2 /dev/nvme0n2p2 # Creating Volume group and physical voulme using single command

sudo pvs /dev/nvme0n2p2

sudo vgs /dev/nvme0n2p2

sudo lvs /dev/nvme0n2p2

sudo lvcreate -n vg2lv2 -L1G vg2
````

````bash                                          
[user1@localhost ~]$ sudo vgcreate vg2 /dev/nvme0n2p2
[sudo] password for user1: 
  Physical volume "/dev/nvme0n2p2" successfully created.
  Volume group "vg2" successfully created

[user1@localhost ~]$ sudo pvs
  PV             VG   Fmt  Attr PSize  PFree 
  /dev/nvme0n1p3 rhel lvm2 a--  58.41g     0 
  /dev/nvme0n2p2 vg2  lvm2 a--  <2.50g <2.50g

[user1@localhost ~]$ sudo vgs
  VG   #PV #LV #SN Attr   VSize  VFree 
  rhel   1   3   0 wz--n- 58.41g     0 
  vg2    1   0   0 wz--n- <2.50g <2.50g

[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g                                                    

[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
└─nvme0n2p3   259:7    0   2.5G  0 part 

[user1@localhost ~]$ sudo lvcreate -n vg2lv2 -L1G vg2
  Logical volume "vg2lv2" created.

[user1@localhost ~]$ sudo lvs
  LV     VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home   rhel -wi-ao----  18.50g                                                    
  root   rhel -wi-ao---- <37.90g                                                    
  swap   rhel -wi-ao----   2.01g                                                    
  vg2lv2 vg2  -wi-a-----   1.00g                                                    

[user1@localhost ~]$ sudo lvcreate -n vg2lv3 -L1G vg2
  Logical volume "vg2lv3" created.

[user1@localhost ~]$ lsblk
NAME           MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0             11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1             11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1        259:0    0    60G  0 disk 
├─nvme0n1p1    259:1    0   600M  0 part /boot/efi
├─nvme0n1p2    259:2    0     1G  0 part /boot
└─nvme0n1p3    259:3    0  58.4G  0 part 
  ├─rhel-root  253:0    0  37.9G  0 lvm  /
  ├─rhel-swap  253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home  253:2    0  18.5G  0 lvm  /home
nvme0n2        259:4    0    10G  0 disk 
├─nvme0n2p1    259:5    0   2.5G  0 part /data
├─nvme0n2p2    259:6    0   2.5G  0 part 
│ ├─vg2-vg2lv2 253:3    0     1G  0 lvm  
│ └─vg2-vg2lv3 253:4    0     1G  0 lvm  
└─nvme0n2p3    259:7    0   2.5G  0 part 
````

**Creating Volume group using 2 Partitions (PVs)**

````bash
[user1@localhost ~]$ sudo vgcreate vg23 /dev/nvme0n2p2 /dev/nvme0n2p3
  Physical volume "/dev/nvme0n2p3" successfully created.
  Volume group "vg23" successfully created

[user1@localhost ~]$ sudo pvs
  PV             VG   Fmt  Attr PSize  PFree 
  /dev/nvme0n1p3 rhel lvm2 a--  58.41g     0 
  /dev/nvme0n2p2 vg23 lvm2 a--  <2.50g <2.50g
  /dev/nvme0n2p3 vg23 lvm2 a--  <2.50g <2.50g

[user1@localhost ~]$ sudo vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  rhel   1   3   0 wz--n- 58.41g    0 
  vg23   2   0   0 wz--n-  4.99g 4.99g

[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g   

[user1@localhost ~]$ sudo lvcreate -n lv23 -L1G vg23
  Logical volume "lv23" created.

[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g                                                    
  lv23 vg23 -wi-a-----   1.00g                                                    

[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ └─vg23-lv23 253:3    0     1G  0 lvm  
└─nvme0n2p3   259:7    0   2.5G  0 part 


[user1@localhost ~]$ sudo lvcreate -n lv24 -L1G vg23
  Logical volume "lv24" created.
[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g                                                    
  lv23 vg23 -wi-a-----   1.00g                                                    
  lv24 vg23 -wi-a-----   1.00g                                                    
[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ ├─vg23-lv23 253:3    0     1G  0 lvm  
│ └─vg23-lv24 253:4    0     1G  0 lvm  
└─nvme0n2p3   259:7    0   2.5G  0 part 


[user1@localhost ~]$ sudo lvcreate -n lv25 -L1G vg23
  Logical volume "lv25" created.
[user1@localhost ~]$ sudo lvs
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home rhel -wi-ao----  18.50g                                                    
  root rhel -wi-ao---- <37.90g                                                    
  swap rhel -wi-ao----   2.01g                                                    
  lv23 vg23 -wi-a-----   1.00g                                                    
  lv24 vg23 -wi-a-----   1.00g                                                    
  lv25 vg23 -wi-a-----   1.00g                                                    
[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ ├─vg23-lv23 253:3    0     1G  0 lvm  
│ └─vg23-lv24 253:4    0     1G  0 lvm  
└─nvme0n2p3   259:7    0   2.5G  0 part 
  └─vg23-lv25 253:5    0     1G  0 lvm  
````

#### Dynamically expanding Logical voulmes

Extending logical volumes (LVs) is most advantages. The extension is possible, if the Volume group (VG) has free space. If VG is full, than add another Physical disk and create (PV).

To extend Volume Group:

vgextend 

To extend Logical volume:

lvextend

To extend Filesystem created on LV:

lvextend -r 
    
    -r option used to resize ext4 or xfs filesystem and respective logical volume


How to change volume group Attr?

sudo vgchange -a y <VG name>  # y for yes


Make filesystem?

mkfs.xfs </dev/VGname/LVname>

or

mkfs.xfs </dev/mapper/VGname-LVname>

LVM:

lvextend -r -l +100%FREE <VGname/LVname>

or

lvextend -r -L +100M <VGname/LVname>


vgextend -v <VGname> <new_partition (/dev/nvme0n2p4)>


**Virtual memory / Swap space**

swapon -s

lvcreate -n <swaplogicalvolume> -L +500m <VGname>

Adding Swap header:

mkswap </dev/VGname/swaplogicalvolume>


swapon -p 10 </dev/VGname/swaplogicalvolume> # p is priority (high value more priority - used)

swapon -s

swapoff -a

Add the below record in **/etc/fstab**

</dev/VGname/swaplogicalvolume>  swap pri=3 0 0

swapon -a

swapon -s



## Module 05 - Creating, Configuring, and Managing FileSystems

In Module 04, We already know about adding new disk and make partitions.

### Creating and manage Local filesystems

- Creating filesystems using (mkfs)
- Read and manage filesystem metadata
- Create and secure mountpoints and mount filesystems
- Extend logical volumes


````bash
# List the block storages
[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ ├─vg23-lv23 253:3    0     1G  0 lvm  
│ └─vg23-lv24 253:4    0     1G  0 lvm  
└─nvme0n2p3   259:7    0   2.5G  0 part 
  └─vg23-lv25 253:5    0     1G  0 lvm  

# List the filesystem info
[user1@localhost ~]$ lsblk -f
NAME          FSTYPE      FSVER            LABEL                    UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sr0           iso9660                      CDROM                    2025-11-09-15-43-20-00                       0   100% /run/media/user1/CDROM
sr1           iso9660     Joliet Extension RHEL-9-6-0-BaseOS-x86_64 2025-04-08-23-13-43-00                       0   100% /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1                                                                                                                   
├─nvme0n1p1   vfat        FAT32                                     9FAF-324C                               591.8M     1% /boot/efi
├─nvme0n1p2   xfs                                                   2cb4ec9f-7d8e-4269-836b-7639df723ba4    603.5M    37% /boot
└─nvme0n1p3   LVM2_member LVM2 001                                  m4GEPW-pZ4s-GPEf-NoNK-sxlq-HzrQ-MANXoC                
  ├─rhel-root xfs                                                   eda35ef0-e7dd-4f9d-835a-7372e55c0207     33.3G    12% /
  ├─rhel-swap swap        1                                         3e536219-7157-40f3-bc18-16139ed667cd                  [SWAP]
  └─rhel-home xfs                                                   6c8f2a47-daa9-441e-8828-916f50aca550     18.3G     1% /home
nvme0n2                                                                                                                   
├─nvme0n2p1   xfs                          DATA                     58187d3a-aaa8-4c34-844d-1d3ef06d446b      2.4G     2% /data
├─nvme0n2p2   LVM2_member LVM2 001                                  UG7KMn-Viz6-ptSM-97m3-aRIQ-PnrW-ki03rR                
│ ├─vg23-lv23 xfs                                                   ec3d9d62-e1e9-4d85-aa9f-07b9df5bb357                  
│ └─vg23-lv24                                                                                                             
└─nvme0n2p3   LVM2_member LVM2 001                                  SPtmLj-tWYr-Sd06-x4fm-1pL3-5C5m-eD4EYw                
  └─vg23-lv25        

# Read Filesystem metadata check isize and bsize
[user1@localhost ~]$ sudo xfs_info /dev/nvme0n2p1
meta-data=/dev/nvme0n2p1         isize=512    agcount=4, agsize=163776 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=655104, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# print label, use -L set set Label (if label not available)
[user1@localhost ~]$ sudo xfs_admin -l /dev/nvme0n2p1
label = "DATA"

# print UUID
[user1@localhost ~]$ sudo xfs_admin -u /dev/nvme0n2p1
UUID = 58187d3a-aaa8-4c34-844d-1d3ef06d446b
                                                                                                     
````

**EXT4**

The below are the most common filesystem types in Linux distributions:

- xfs
- ext4

````bash
mkfs.xfs <PartitionName or DiskName>

mkfs.ext4 <PartitionName or DiskName>

# Reading ext4 file metadata
sudo dumpe2fs /dev/<partitionname>

# Create Label, incase forgot while mkfs time
sudo tune2fs -L "<labelname>" /dev/<partitionname>  
````

Note:

Important Options - Avoid Unwanted filesystem checks

- Check interval: The internal between filesystem checks. the filesystem interval will always expire when you need the system the quickest

- Maximum mount count: When the filesystem reaches the max count value the filesystem is automatically checked on boot



**Securing Mount Points**

Before mounting the disk or disk partition to the directory (ex: /data) in the root Filesystem (/) is not accessable to the users except root. so

sudo mkdir /data # **this a directory inside the root filesystem, so root user only need to access the directory until mount to disk partition or disk**

Before mount the directory permissions:

sudo chmod -v 700 /data

After mount:

mount <disk partition> /data

sudo chmod -v 707 /data # its depended on the our usecase

The real magic is:

when you **umoumt** and try to access the /data, then you will get **permission error**. Becuase, **Automatically inherit the permissions** depend on the mount status.

sudo umount /data

sudo ls -l /

drwx------.   2 root root    6 Nov 14 18:37 data     # 700

cd /data # permission error

sudo mount <partitionfullpath> /data

sudo ls -l /

cd /data # No error

````bash
[user1@localhost ~]$ sudo mount /dev/nvme0n2p1 /data

[user1@localhost ~]$ ls -l /

drwx---rwx.   2 root root    6 Nov 14 18:25 data  # 707
````

Example:

````bash
[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part 
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ ├─vg23-lv23 253:3    0     1G  0 lvm  
│ └─vg23-lv24 253:4    0     1G  0 lvm  
├─nvme0n2p3   259:7    0   2.5G  0 part 
│ └─vg23-lv25 253:5    0     1G  0 lvm  
└─nvme0n2p4   259:8    0   1.8G  0 part 

# The directory permissions before mounting
[user1@localhost ~]$ sudo chmod -v 700 /data
mode of '/data' changed from 0755 (rwxr-xr-x) to 0700 (rwx------)

# No one can"t access the directory except root
[user1@localhost ~]$ cd /data
bash: cd: /data: Permission denied

# mount the disk to the mount point (directory)
[user1@localhost ~]$ sudo mount /dev/nvme0n2p1 /data
[user1@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/user1/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/user1/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    60G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  58.4G  0 part 
  ├─rhel-root 253:0    0  37.9G  0 lvm  /
  ├─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
  └─rhel-home 253:2    0  18.5G  0 lvm  /home
nvme0n2       259:4    0    10G  0 disk 
├─nvme0n2p1   259:5    0   2.5G  0 part /data
├─nvme0n2p2   259:6    0   2.5G  0 part 
│ ├─vg23-lv23 253:3    0     1G  0 lvm  
│ └─vg23-lv24 253:4    0     1G  0 lvm  
├─nvme0n2p3   259:7    0   2.5G  0 part 
│ └─vg23-lv25 253:5    0     1G  0 lvm  
└─nvme0n2p4   259:8    0   1.8G  0 part 

# Changing the permisions after mounting
[user1@localhost ~]$ sudo chmod -v 707 /data
mode of '/data' changed from 0755 (rwxr-xr-x) to 0707 (rwx---rwx)

# Access checking
[user1@localhost ~]$ cd /data
[user1@localhost data]$ cd -
/home/user1

# Unmount and access checking
[user1@localhost ~]$ sudo umount /data
[user1@localhost ~]$ cd /data
bash: cd: /data: Permission denied 
````

**Extending Logical Volumes**

This topic already covered in the module 04 - Dynamically expanding Logical voulmes



### Special filesystem permissions for Collaborative 

#### Manage directory permissions for collaboration

**Special Permissions**

stat -c %a /etc/hosts

0644

 s   u   g   o
--- --- --- --- => 4 blocks * 3 bits = 12 bits


We already know, 644 repersents the permissions of user, group, and others

now, we are learning about **1st block** -> special permissions

SUID - 4 - Used on programs to run as the user owner during execution

SGID - 2 - On directories, new files are assigned the group owner from the directory

Sticky Bit - 1 - With this set users can only delete files they own from shared directories

````bash
[pavan@localhost ~]$ mkdir -p prems/dir{1..4}
[pavan@localhost ~]$ ls -l prems
total 0
drwxr-xr-x. 2 pavan pavan 6 Nov 23 01:34 dir1
drwxr-xr-x. 2 pavan pavan 6 Nov 23 01:34 dir2
drwxr-xr-x. 2 pavan pavan 6 Nov 23 01:34 dir3
drwxr-xr-x. 2 pavan pavan 6 Nov 23 01:34 dir4

# Special permissions
[pavan@localhost ~]$ chmod -v 1777 prems/dir1 # Sticky bit set
mode of 'prems/dir1' changed from 0755 (rwxr-xr-x) to 1777 (rwxrwxrwt)

[pavan@localhost ~]$ chmod -v 2777 prems/dir2 # SGID bit set
mode of 'prems/dir2' changed from 0755 (rwxr-xr-x) to 2777 (rwxrwsrwx)

[pavan@localhost ~]$ chmod -v 3777 prems/dir3 # Both the sticky bit and SGID bit set
mode of 'prems/dir3' changed from 0755 (rwxr-xr-x) to 3777 (rwxrwsrwt)

[pavan@localhost ~]$ chmod -v 1770 prems/dir4 # Sticky bit is set but no permissions to others
mode of 'prems/dir4' changed from 0755 (rwxr-xr-x) to 1770 (rwxrwx--T)
[pavan@localhost ~]$ ls -l prems
total 0
drwxrwxrwt. 2 pavan pavan 6 Nov 23 01:34 dir1
drwxrwsrwx. 2 pavan pavan 6 Nov 23 01:34 dir2
drwxrwsrwt. 2 pavan pavan 6 Nov 23 01:34 dir3
drwxrwx--T. 2 pavan pavan 6 Nov 23 01:34 dir4

(or) 

find ~/prems/ -type d -perm /g=s,o=t # List dirs where either SGID or Sticky bit set

type -> d for directories, f for regular files, l for linked files

d -> for directories

g (group) -> SGID

o (others) -> t -> sticky

/ -> either, - -> both

[pavan@localhost ~]$ find prems -type d 
prems
prems/dir1
prems/dir2
prems/dir3
prems/dir4

[pavan@localhost ~]$ find prems -type d -perm /g=s
prems/dir2
prems/dir3

# List dirs where either SGID nor Sticky bit set 
[pavan@localhost ~]$ find prems -type d -perm /g=s,o=t
prems/dir1
prems/dir2
prems/dir3
prems/dir4

# List dirs where both SGID or Sticky bit set 
[pavan@localhost ~]$ find prems -type d -perm -g=s,o=t
prems/dir3

# List dirs where Sticky bit set or world writable
[pavan@localhost ~]$ find prems -type d -perm /o=tw
prems/dir1
prems/dir2
prems/dir3
prems/dir4
````

**Sticky bit example:**

````bash
# Switch to root user
[pavan@localhost ~]$ sudo -i

# Create user2
[root@localhost ~]# useradd -m user2
[root@localhost ~]# passwd user2
Changing password for user user2.
New password: 
BAD PASSWORD: The password contains the user name in some form
Retype new password: 
passwd: all authentication tokens updated successfully.

# Create a new group and add user2 to the created group
[root@localhost ~]# sudo groupadd devops
[root@localhost ~]# gpasswd -a user2 devops
Adding user user2 to group devops

# Adding user2 to wheel group limited time
echo "gpasswd -a user2 wheel" | at now + 120 minutes

# Switch user to user2
[root@localhost ~]# su - user2
[user2@localhost ~]$ pwd
/home/user2


# Create a directory
[user2@localhost ~]$ sudo mkdir -p -m 700 /teams/devops # Before mounting, the root user only access the /teams/devops directory
# Observe the group and user details
[user2@localhost ~]$ ls -l /teams/devops
drwx------. 2 root root 6 Nov 23 02:31 devops

drwxr-xr-x.   3 root root   20 Nov 23 02:31 teams

# Access checking for User2
[user2@localhost ~]$ cd /teams
[user2@localhost teams]$ cd devops
-bash: cd: devops: Permission denied

[user2@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/pavan/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/pavan/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    50G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  48.4G  0 part 
  ├─rhel-root 253:0    0  46.4G  0 lvm  /
  └─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
nvme0n2       259:4    0    20G  0 disk 
└─nvme0n2p1   259:5    0    10G  0 part 

# Mounting
[user2@localhost ~]$ sudo mount /dev/nvme0n2p1 /teams/devops

[user2@localhost ~]$ lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0            11:0    1 167.3M  0 rom  /run/media/pavan/CDROM
sr1            11:1    1  11.9G  0 rom  /run/media/pavan/RHEL-9-6-0-BaseOS-x86_64
nvme0n1       259:0    0    50G  0 disk 
├─nvme0n1p1   259:1    0   600M  0 part /boot/efi
├─nvme0n1p2   259:2    0     1G  0 part /boot
└─nvme0n1p3   259:3    0  48.4G  0 part 
  ├─rhel-root 253:0    0  46.4G  0 lvm  /
  └─rhel-swap 253:1    0     2G  0 lvm  [SWAP]
nvme0n2       259:4    0    20G  0 disk 
└─nvme0n2p1   259:5    0    10G  0 part /teams/devops

# Change group and modify permissions for the filesystem (/teams/devops)
[user2@localhost ~]$ sudo chgrp devops /teams/devops
[user2@localhost ~]$ sudo chmod -v 770 /teams/devops
mode of '/teams/devops' changed from 0755 (rwxr-xr-x) to 0770 (rwxrwx---)

[user2@localhost ~]$ ls -l /teams
total 0
drwxrwx---. 2 root devops 6 Nov 23 02:13 devops


# Check the user belong to devops group or not
[user2@localhost devops]$ id
uid=1001(user2) gid=1001(user2) groups=1001(user2),10(wheel),1002(devops) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

## Before Special permissions

[user2@localhost ~]$ cd /teams/devops
[user2@localhost devops]$ touch file1
[user2@localhost devops]$ sudo touch root1
[user2@localhost devops]$ ls -l
total 0
-rw-r--r--. 1 user2 user2 0 Nov 23 02:54 file1
-rw-r--r--. 1 root  root  0 Nov 23 02:54 root1

[user2@localhost devops]$ rm *
rm: remove write-protected regular empty file 'root1'? yes
[user2@localhost devops]$ ls -l
total 0

## After Special permissions

[user2@localhost ~]$ sudo chmod o+t /teams/devops

[user2@localhost ~]$ ls -l /teams/
total 0
drwxrwx--T. 2 root devops 6 Nov 23 02:58 devops

[user2@localhost ~]$ touch /teams/devops/file1
[user2@localhost ~]$ sudo touch /teams/devops/root1

[user2@localhost ~]$ ls -l /teams/devops
total 0
-rw-r--r--. 1 user2 user2 0 Nov 23 03:03 file1
-rw-r--r--. 1 root  root  0 Nov 23 03:03 root1

[user2@localhost ~]$ cd /teams/devops

[user2@localhost devops]$ rm root1
rm: remove write-protected regular empty file 'root1'? yes
rm: cannot remove 'root1': Operation not permitted

[user2@localhost devops]$ rm file1

[user2@localhost devops]$ ls -l
total 0
-rw-r--r--. 1 root root 0 Nov 23 03:03 root1

````

Note:

rm root1 # Error: operation not permitted, because sticky bit won't allow user2 to delete file. Because he is not the file owner.

rm file1 # No Error, because user2 is the file owner

*Sticky bit permission at directory level - only allow the file owner delete the files inside that directory*


**SGID**

````bash
su - user2

cd /teams/devops

[user2@localhost devops]$ umask 007

[user2@localhost devops]$ touch file1

[user2@localhost devops]$ sudo touch root1

[user2@localhost devops]$ ls -l
total 0
-rw-rw----. 1 user2 user2 0 Nov 23 03:18 file1
-rw-r-----. 1 root  root  0 Nov 23 03:18 root1

# Adding SGID (. -> /teams/devops)
[user2@localhost devops]$ sudo chmod -v g+s .
mode of '.' changed from 1770 (rwxrwx--T) to 3770 (rwxrws--T)

# Removing sticky bit
[user2@localhost devops]$ sudo chmod -v o-t .
mode of '.' changed from 3770 (rwxrws--T) to 2770 (rwxrws---)

[user2@localhost teams]$ ls -l /teams

drwxrws---. 2 root devops 32 Nov 23 03:18 devops

[user2@localhost devops]$ touch file2
[user2@localhost devops]$ ls -l
total 0
-rw-rw----. 1 user2 user2  0 Nov 23 03:18 file1
-rw-rw----. 1 user2 devops 0 Nov 23 03:24 file2

Note: **observe the user and group difference between file1 and file2** - The directory level group assigned automatically to the newly created files/directories inside /teams/devops after SGID 

# Switch to root user
[user2@localhost devops]$ sudo -i

[root@localhost ~]# umask 007
[root@localhost ~]# touch /teams/devops/root2
[root@localhost ~]# exit
logout
[user2@localhost devops]$ ls -l
-rw-r-----. 1 root  root   0 Nov 23 03:18 root1
-rw-rw----. 1 root  devops 0 Nov 23 03:29 root2

Note: **observe the user and group difference between root1 and root2** - directory level user group assigned automatically to new files/directories inside /teams/devops, by any user belong to devops group or root user.

````

Note:

The special permission are very useful to work collaboratively with consistency (automatically same group name), security (only file owner delete files).


#### Sharing filesystems using NFS

We can share the files using NFS protocol between client and server systems. 

+ Install nfs-utils in both server and client VMs
+ Inside server VM, add nfs service in Firewall inbound rules
+ Inside server VM, edit **/etc/exports** default configuration file or **/etc/exports.d/<customname>.exports**


- **nfsconf** management tool that writes to the new **/etc/nfs.conf** configuration file.

  nfsconf --set nfsd vers4 y # this command will modify in /etc/nfs.conf, y -> enable, n -> disable

  nfsconf --set nfsd tcp y

  nfsconf --set nfsd udp n

  nfsconf --set nfsd vers3 n

- managing the inbound connections using firewalld

  firewall-cmd --state # status check

  firewall-cmd --list-all

  firewall-cmd --add-service=nfs

  firewall-cmd ----runtime-to-permanent

Note: 

For NFS, we need two VMs one act as **server** and other act as **client**. In both VMs we need to install **nfs-utils** package

By default, Firewall **blocks the nfs service** on server VM. On the NFS server we can allow inbound **TCP port 2049 or NFS service** by making use of the NFS service XML file.

````bash
# Inside server VM

# Check Open ports in VM before installing nfs-utils
[user1@localhost ~]$ ss -ntl
State         Recv-Q                       Send-Q                Local Address:Port                  Peer Address:Port                       
LISTEN        0                            4096                     127.0.0.1:631                        0.0.0.0:*                          
LISTEN        0                            128                      0.0.0.0:22                           0.0.0.0:*                          
LISTEN        0                            4096                     [::1]:631                              [::]:*                          
LISTEN        0                            128                      [::]:22                                [::]:*   



# 01 - Install nfs-utils
sudo yum install nfs-utils -y

# Start and enable nfs service
sudo systemctl enable --now nfs-server

sudo systemctl status nfs-server

# Check the allowed services in the firewall
[user1@localhost ~]$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens160
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

# Check Open ports in VM after installing nfs-utils and starting the service
root@localhost ~]# ss -ntl
State                 Recv-Q                Send-Q                     Local Address:Port                    Peer Address:Port                      
LISTEN                  0                    4096                        0.0.0.0:50085                            0.0.0.0:*                         
LISTEN                  0                    4096                        0.0.0.0:20048                            0.0.0.0:*                         
LISTEN                  0                    4096                        127.0.0.1:631                            0.0.0.0:*                         
LISTEN                  0                    128                         0.0.0.0:22                               0.0.0.0:*                         
LISTEN                  0                    4096                        0.0.0.0:2049                             0.0.0.0:*                         
LISTEN                  0                    4096                        0.0.0.0:111                              0.0.0.0:*                         
LISTEN                  0                    64                          0.0.0.0:41143                            0.0.0.0:*                         
LISTEN                  0                    4096                          [::]:57223                               [::]:*                         
LISTEN                  0                    4096                          [::]:20048                               [::]:*                         
LISTEN                  0                    4096                          [::1]:631                                [::]:*                         
LISTEN                  0                    64                            [::]:39321                               [::]:*                         
LISTEN                  0                    128                           [::]:22                                  [::]:*                         
LISTEN                  0                    4096                          [::]:2049                                [::]:*                         
LISTEN                  0                    4096                          [::]:111                                 [::]:*        


# 02 - add nfs service in firewall

sudo firewall-cmd --list-all

sudo firewall-cmd --permanent --add-service=nfs

sudo firewall-cmd --list-all

# 03 - /etc/exports.d/teams.exports

# Creating shared filesystem
sudo mkdir -p -m 700 /teams/devops # only root user access the directory

# Adding files inside shared directory
sudo find /usr/share/doc -name '*.pdf' -exec sudo cp {} /teams/devops \;

sudo ls -l /teams/devops

# Create the custom configuration file and enter
sudo vim /etc/exports.d/teams.exports

  /teams/devops <Public IP>/<CIDR>(no_root_squash,rw,sync)
  # shared directory <who will access the directory> <permission and sync, no_root_squash>

sudo exportfs -r

sudo exportfs -rav

[user1@localhost ~]$ sudo exportfs

/teams/devops 192.168.28.128/24

````


````bash
# Inside Client VM

# 01 - Install nfs-utils
sudo yum install nfs-utils -y


# 02 - mount nfs to clent side filesystem

sudo mount -t nfs4 <NFS server VM IP>:/teams/devops /mnt
# sudo mount -t nfs4 -o vers=4 <server-ip>:/teams/devops /mnt

sudo ls -l /mnt # check you are able to access the files, you can't able to list because (in server side, sudo mkdir -p -m 700 /teams/devops)

  # Before mounting 

  ## root user
  [root@localhost ~]# ls -l /
  drwxr-xr-x.   2 root root  150 Nov 21 11:07 mnt

  [root@localhost ~]$ cd /mnt

  [root@localhost mnt]$ 

  # Normal user (wheel)
  [pavan@localhost ~]$ cd /mnt

  [pavan@localhost mnt]$ 


  # After mounting

  ## root user
  [root@localhost ~]# ls -l /
  drwx------.   2 root root  150 Nov 21 11:07 mnt

  [root@localhost ~]# cd /mnt
  -bash: cd: /mnt: Permission denied

  Note: After mounting, the directory /mnt permissions reflect server-side ownership, not client-side permissions.
````

**About no_root_squash**

Inside NFS server VM

sudo vim /etc/exports.d/teams.exports

  /teams/devops <Public IP>/<CIDR>(rw,sync)

Without `no_root_squash`, even the **root user** on the client cannot fully access the exported directory (`/mnt`) on the NFS server.


Why this happens:

1. **NFS user mapping**

   By default, NFS does **not trust the client’s root user**. Instead, it maps requests from **root (UID 0) on the client** to **nobody:nogroup** on the server. This is called **root squashing**.

   * Example:

     ```text
     Client root (UID 0) → Server nobody (UID 65534)

     [user1@localhost teams]$ sudo cat /etc/passwd | grep -i nobody
   
     nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
     ```
   * This prevents a remote root from having full control over the exported filesystem, which is a **security feature**.

2. **Effect on permissions**

  Because the client’s root is mapped to `nobody`, it loses write access (or sometimes even read access) if the permissions on the exported directory are restrictive. So even `root` on the client behaves like an unprivileged user on the server.

3. **`no_root_squash`**

   Adding this option in `/etc/exports.d/teams.exports` disables the root mapping:

   ```text
   /teams/devops <Public IP>/<CIDR>(rw,sync,no_root_squash)
   ```

   * Now, **root on the client remains root on the server**, with full privileges.
   * This is **dangerous** if the client is not fully trusted, because it gives root full control over the server’s filesystem.


Security implications:

* `no_root_squash` should only be used in **trusted environments** (like your own internal lab network).


✅ **Summary:**

* Default behavior: `root` on the client → `nobody` on the server (**root squashed**)
* With `no_root_squash`: `root` on the client → `root` on the server (**full access**)
* Without `no_root_squash`, root access fails if directory permissions don’t allow `nobody` to write.


NFS Server VM:

````bash
[user1@localhost teams]# sudo chgrp -v devops devops
changed group of 'devops' from root to devops

[user1@localhost teams]# ls -l
total 0
drwxr-xr-x. 2 root devops 150 Nov 21 17:07 devops

Note down the group ID of the shared file belong to:  **1001(devops)**  


[user1@localhost teams]$ sudo chmod -v 1770 devops
mode of 'devops' changed from 0755 (rwxr-xr-x) to 1770 (rwxrwx--T)

[user1@localhost teams]$ ls -l
total 0
drwxrwx--T. 2 root devops 150 Nov 21 17:07 devops
````

NFS Client VM:

````bash
[pavan@localhost /]$ ls -l
drwxrwx--T.   2 root devops  150 Nov 21 11:07 mnt

[pavan@localhost /]$ id
uid=1000(pavan) gid=1000(pavan) groups=1000(pavan),10(wheel),1001(devops) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

[pavan@localhost /]$ sudo -i
[root@localhost ~]# cd /mnt
-bash: cd: /mnt: Permission denied
````

Note:

```
/etc/exports.d/teams.exports → /teams/devops <Public IP>/<CIDR>(rw,sync)
```

Because `no_root_squash` is **not** used, the **root user on the client** is mapped to `nobody:nogroup` on the NFS server:

```bash
[user1@localhost teams]$ sudo cat /etc/passwd | grep -i nobody
nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
```

However, the **devops** group has the same **GID 1001** on both the server and the client.
This means the client user **pavan**, who belongs to the `devops` group, receives proper **group-level permissions** on the shared directory:

```
drwxrwx--T. 2 root devops 150 Nov 21 17:07 devops
```



````bash 

Inside Client VM 

## mount persistance

- edit /etc/fstab

  <NFS server VM IP>:/teams/devops /mnt nfs defaults 0 0

or 

- autofs

autofs service can mount these exports automatically for you when needed. 

# Install autofs on client server
sudo yum install autofs

sudo systemctl enable --now autofs

# edit /etc/auto.master or create custom master configuration file ex: /etc/auto.master.d/teams.autofs
sudo vim /etc/auto.master.d/teams.autofs

  # enter top level directory,  sub-level directory configuration file path
  /teams /etc/auto.teams # this create /teams directory automatically in client server

# Inside /etc/auto.teams

sudo vim /etc/auto.teams

  devops -rw,soft <NFS server VM IP>:/teams/devops # this create /teams/devops directory automatically in client server

results:

## root user
[root@localhost]# ls -l /
drwxr-xr-x.   2 root root    0 Nov 25 11:58 teams

[root@localhost]# cd /teams
[root@localhost teams]# cd devops
-bash: cd: devops: Permission denied

## normal user belong to devops group
[root@localhost teams]# su - pavan
[pavan@localhost ~]$ cd /teams/devops
[pavan@localhost devops]$ ls -l
total 2272
-rw-r--r--. 1 root root 691531 Nov 21 11:07 gutenprint-users-manual.pdf
-rw-r--r--. 1 root root 235941 Nov 21 11:07 Padauk-features.pdf
-rw-r--r--. 1 root root 371473 Nov 21 11:07 Padauk-typesample.pdf
-rw-r--r--. 1 root root 995234 Nov 21 11:07 PakTypeNaskhBasicFeatures.pdf
-rw-r--r--. 1 root root  24867 Nov 21 11:07 pigz.pdf

# Check the group ID of the /teams/devops directory on the server VM
# Inside client VM, Make sure to create group with same group ID add the user to the group

sudo groupadd devops # creating a new group

sudo gpasswd -a pavan devops # adding the pavan to devops group

````
Advantages of using NFS with autofs:
- only mount the directory when its needed, so **reduce network traffic and load on server**


Server VM:

sudo vim /etc/exports.d/teams.exports

  /teams/devops 172.31.0.0/16(rw,sync)

  172.31.0.0/16 -> VPC 


AWS: (client vm)

[ec2-user@ip-172-31-79-175 ~]$ sudo yum install nfs-utils

[ec2-user@ip-172-31-79-175 ~]$ id
uid=1000(ec2-user) gid=1000(ec2-user) groups=1000(ec2-user),4(adm),190(systemd-journal),1001(devops) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023


[ec2-user@ip-172-31-79-175 ~]$ sudo vim /etc/auto.master.d/teams.autofs

  /teams /etc/auto.teams

[ec2-user@ip-172-31-79-175 ~]$ sudo vim /etc/auto.teams

  devops -rw,soft 192.168.28.129:/teams/devops

192.168.28.129  -> Server VM IP

[ec2-user@ip-172-31-79-175 ~]$ sudo systemctl restart autofs
[ec2-user@ip-172-31-79-175 ~]$ sudo systemctl status autofs
● autofs.service - Automounts filesystems on demand
     Loaded: loaded (/usr/lib/systemd/system/autofs.service; disabled; preset: disabled)
     Active: active (running) since Tue 2025-11-25 17:23:13 UTC; 9s ago
 Invocation: 73617b064a474d70bb925ba0e2f92245
   Main PID: 2030 (automount)
      Tasks: 7 (limit: 5108)
     Memory: 1.9M (peak: 2.5M)
        CPU: 23ms
     CGroup: /system.slice/autofs.service
             └─2030 /usr/sbin/automount --systemd-service --dont-check-daemon

Nov 25 17:23:13 ip-172-31-79-175.ec2.internal systemd[1]: Starting autofs.service - Automounts filesystems on demand...
Nov 25 17:23:13 ip-172-31-79-175.ec2.internal (utomount)[2030]: autofs.service: Referenced but unset environment variable evaluates to an empty string: OPTIONS
Nov 25 17:23:13 ip-172-31-79-175.ec2.internal systemd[1]: Started autofs.service - Automounts filesystems on demand.


drwxr-xr-x. 2 root root 0 Nov 25 17:23 teams
[ec2-user@ip-172-31-79-175 /]$ cd teams/ 
[ec2-user@ip-172-31-79-175 teams]$ ls -l 
total 0 
[ec2-user@ip-172-31-79-175 teams]$ cd devops 
-bash: cd: devops: No such file or directory

Problem Reason:

NETWORKING

➡ These networks cannot reach each other directly.
➡ AWS EC2 cannot reach your on-prem VMware NFS server.

172.31.0.0/16 (AWS VPC range)

VMware: 192.168.28.0/24

AWS VPC is a completely isolated virtual network inside Amazon.

Your VMware network is a private LAN inside your home/office.

solutions:

Option 1 — AWS Site-to-Site VPN


Option 2 — WireGuard VPN (Easiest)

Install WireGuard:

  On AWS EC2

  On VMware VM



Additional Info:

````text
+ NFSv4 uses a single port (2049), so firewall rules are simpler than NFSv3

+ If only NFSv4 is needed, you can disable NFSv3 explicitly in /etc/nfs.conf

+ sync ensures writes are committed immediately; safer but slightly slower.

+ async is faster but risky if server crashes.

+ soft vs hard in mounts: soft allows client to fail on server unavailability; hard retries indefinitely.

If SELinux is enabled on server, may need:

  sudo setsebool -P nfs_export_all_rw 1

  sudo chcon -Rt nfs_t /teams/devops
````

### VDO

VDO (Virtual Data Optimizer). It is Logical abstraction layer between filesystem and physical storage.

````bash
# install VDO and kernel module
sudo yum install vdo kmod-kvdo

# enable and start VDO service
sudo enable --now vdo.service

# module loaded
modprobe kvdo

# create VDO, /dev/disk > 4G
sudo vdo create --name=vdo1 --device=/dev/<disk or disk partition> --vdoLogicalSize=20G

# Check deduplication and compression is enabled or not
sudo vdo status --name=vdo1 | grep -E '(Dedup|Compression)'

# incase of disable
sudo vdo enableDeduplication --name=vdo1
sudo vdo enableCompression --name=vdo1

# make filesystem
sudo mkfs.xfs -K /dev/mapper/vdo1

sudo mkdir -m 700 -p /teams/vdo # only root user can access

# persistant mount
sudo vim /etc/fstab
  # condition: vdo service must running before mounting
  /dev/mapper/vdo1 /teams/vdo xfs x.systemd.requires=vdo.service 0 0

sudo mount -a

sudo chgrp devops /teams/vdo


# check 
mount -t xfs

chmod -v 3770 /teams/vdo # change permissions speacial and group

sudo cp /usr/share/doc/*.html /teams/vdo/

for i in {1..5} ; do sudo cp /usr/share/doc/*.html /teams/vdo/file{i}; done


# check vdo stats
vdostats --human-readable

du -sh /teams/vdo


# Increase the Logical size

sudo vdo growLogical --name=vdo1 --vdoLogicalSize=40G

vdo status --name=vdo1 | grep -i 'Logical size'

du -sh /teams/vdo # check the size in filesystem

xfs_growfs /dev/mapper/vdo1

du -sh /teams/vdo # check the size in filesystem
````

VDO’s main purpose is space efficiency (dedup + compression). It is NOT a volume manager like LVM/Stratis.


### Layered storage using Stratis

Stratis = Next-generation volume manager

**Stratis**
stratis volume management - managing volumes with stratis allows you to create **thinly provisioned volumes and filesystems with a single command** whilst utilizing existing dev-mapper and XFS technology.

**Managing Stratis Pools**
Stratis pools **aggregate storage space** and **represent volume groups and thin pools** in Device Mapper (DM) management. The sub-command **add-data** is used to extend the size of an existing pool.

````bash
# 01 - Install stratis and CLI tool
sudo yum install stratisd stratis-cli

sudo systemctl enable --now stratisd

# 02 - pool creation
sudo pool create pool1 /dev/<disk or disk partition>

# extend pool size
sudo pool add-data pool1 /dev/<another disk or partition>

sudo stratis pool list

# 03 - Create filesystem in stratis
sudo stratis filesystem create pool1 fs1

mkdir -p -m 700 /teams/stratis

mount /stratis/pool1/fs1 /teams/stratis

sudo chgrp devops /teams/stratis

sudo chmod -v 3770 /teams/stratis

# to mount persistance
sudo vim /etc/fstab

/stratis/pool1/fs1 /teams/stratis x.systemd.requires=stratisd.service 0 0

mount -a

mount -t xfs
````

**Snapshot**
sudo stratis filesystem snapshot pool1 fs1 snap1

sudo mkdir /backup

mount /stratis/pool1/sanp1 /backup # useful in emergency time

umount /backup

sudo stratis filesystem destroy pool1 snap1



Questions?

VDO and Stratis comparsion:

vdo is an abstract layer between the filesystem and physical storage.

✔️ VDO

- You create a VDO device on top of a disk

- Specify a logical size larger than the disk

- Format it with a filesystem

- Mount it

- VDO provides deduplication, compression, thin provisioning


stratis create a pool between the filesystem and physical storage.

✔️ Stratis

- You create a pool from one or more disks

- You create filesystems inside the pool

- Filesystems are thin-provisioned

- Stratis provides snapshots, easier management, pooling


### SELinux and NFS onfiguration

**SELinux support for NFSv4**

Installing NFS man pages

Man pages for SELinux types can be installed using the command **sepolicy**

````bash
# Install
sudo yum whatprovides "*/sepolicy" # check what package sepolicy comes from?
sudo yum install <packagename>

sepolicy manpage -d nfsd_t -p /usr/share/man/man8

# update database
mandb

# Search for
apropos _selinux 

# read man pages
man nfsd_selinux
man 8 nfsd_selinux
````
---

## Module 06 - Depoly, Configuring and Maintaining Systems

### Managing Software Packages


### Configuring Time Services

What I am learning under this section?

- Network time protocol (difference between chronyd and ntpd)
- chrony
- timedatectl command and configuring local time
- editing files with sed command

#### chrony

chrony is the package comes with server and client.

yum list chrony # check chrony installed or not

systemctl status chronyd # check the chronyd service status

**chrony configuration**

/etc/chrony.conf in RHEL 

/etc/chrony/chrony.conf in Ubuntu

Editing the configuration may be useful to set a local timeserver source provided by your own network infrastructure or choosing an external pool based on geography.

**sed - stream editer**

cat /etc/chrony.conf

man 5 chrony.conf # detail documentation

vim my.sed
````bash
# d for delete, commented lines and empty lines
/^(#|$)/d
# edit the line start with pool 
/^pool/i pool uk.pool.ntp.org
/^pool.*rhel/d
````

sed -Ef my.sed /etc/chrony.conf # E for enhanced regular expresion. it will not edit the file, just shows the how it modifies the file

sed -i -Ef my.sed /etc/chrony.conf # apply the modifications by editing (i for inplace edit)

systemctl restart chrony

**chronyc tools**

(chronyc -> for client, chronyd -> for daemon)

chrony advantages:
- Fast synchronization
- Cater for CPU load

Tools:
  - chronyc tracking # command
  - chronyc sources # command ... other command available (chronyc sources -v)
  - chronyc


### Managing Systemd targets

what I am learning in this section?

- The purpose of systemd targets
- Identify the current targets
- Change targets on running system
- Modify the default target
- Booting to a specified target
- Modify the bootloader GRUB

**systemd targets**
targets was introduced with RHEL 7 in systemd. In previous versions runlevels were used.

**targets represent groups of services that should be loaded** and replace runlevels which were used previously. Unlike runlevels, **targets have descriptive names such as graphical and multi-user**.

runlevel

man runlevel # old one

systemctl list-units --type target --state active # current target

systemctl get-default

systemctl cat multi-user.target

systemctl set-default graphical.target # change the deafult target

systemctl isolate # useful to change the target

systemctl isolate graphical.target

runlevel 



