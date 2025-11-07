Day 1:

Install RHEL
-------------
- using ISO, developer account subscription
- using Vagrant
 
Connect to RHEL using SSH
--------------------------
In real time, remote login into the VM using the SSH. mostly using username and password, sometimes username and private key.


---

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
cat > file1 << EOF
Line 1
Line 2
EOF

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
cat << EOF | sudo tee /etc/motd
Welcome to the server!
Authorized access only.
EOF

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
````bash
		cat softlink1
		cat: softlink1: No such file or directory
````

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




**User and Group Management**

````bash
sudo usermod -aG wheel user1 # adding user1 to wheel group

id user1
````

Note: If the group details are not updated, than Logout and Login to see the changes

Create User and group
---------------------

````bash
# creating user with home directory (/home/alice)
sudo useradd -m alice
sudo passwd alice

# creating group
sudo groupadd devops
# adding user (alice) to group (devops)
sudo usermod -aG devops alice
````


switch groups - sg, newgrp:
------------------------------

touch file1

ls -l file1

newgrp wheel # opens new shell
or
sg wheel

touch file 2

ls -l file2

exit # return to previous shell

Check the file1 and file2 group names


Switch users:
-------------

su  (switched to the root user, your working directory won't change)

The below both commands gives a full login shell and start in the root user's home directory

su - 

su -l 

su - user1

Archiving Files in Linux
=========================
- write-only files
- securing directories
- archiving files using tar and star
- file compression using gzip and bzip2


write-only files
----------------
touch log

ls -l log

chmod -v u=w log  (u - user)

ls -l log

cat log (permision denied, because user have only write permision)

echo "new line added" >> log

sudo cat log 


Securing directories
--------------------

mkdir -m 155 project1 (m - mode, 1 for user, 5 for group, 5 for others)

ls project1


cd project1

ls (permision denied)

cd ..

chmod -v u=wx project1

echo "new information into new file" > project1/file1

cat project1/file1   (this command works, because the user have execute permissions)

ls project1 (permission denied, because user doesn't have read permision)

Where it is useful?
The user know the files in the directory and he execute the files in the directory.
secure because no one can see the files inside the directory.

tar
---

tar - Tape Archives

tar can be used to create file archives. 

tar file is not compressed but may appear to be a slightly small size than the original content. this is due to the more efficient use of blocks in the filesystem.


sudo du -sh /etc (du - disk usage, s - size of summary)

sudo tar -cf etc.tar /etc 

ls -lh etc.tar


-c for create

-t table of contents

-x extract

-f archive file


star
----
sudo yum install -y star


file compression
----------------
2 utilites
	gzip / gunzip
	bzip2 / bunzip2
	

tar -czf (z for gzip)

tar -cjf (j for bzip2)


example:

sudo tar -cf etc.tar /etc

ls -lh etc.tar

time gzip etc.tar

ls -lh

to expand the gzip file (unzip)

	gunzip etc.tar.gz
	


time bzip2 etc.tar

bunzip2 etc.tar.bz2 # unzip




=============Module 2============


Bash/Shell Scripting
====================

overview
--------
- working with bash exit codes and simple logic
- creating scripts and processing arguments
- reading user input during execution
- using logic and looping syntax
- using functions to improve your code

-------bash exit codes and simple logic------
Traditional CLI commands for loops

	for i in Hallo pavan Tuschss; do echo $i; done
	
		note: Hallo pavan Tuschuss is list with size 3
	
	i=4 ; while [ $i -gt 0 ]; do echo $i; let i-=i; done
	

How to know the type?

	type for  # this provides the type information about the for


example:
Create 3 users with home directories?

	for user in pavan ram hari; do sudo useradd -m $user; echo Password1 | sudo passwd --stdin $user; tail -n1 /etc/passwd; done

Delete 3 users and also their home directories?

	for user in pavan ram hari; do sudo userdel -r $user; tail -n3 /etc/passwd; done


Exit Codes
-----------
0 -> is success


variable-> $? -> provides recent command exit code
echo $?

getent passwd bob ; echo $? # if user exit than value 0


simple logic
-------------
There was explantion behind the "simple logics" used in the loops.
	&& -> used to combine two commands, but we need to understand "The second command only runs if the first command succeeds.
	mkdir dir1 && cd dir1
	
	|| -> the second command only runs if the first command fails.
	cd dir || mkdir dir1

example:

cd mkt || mkdir mkt && cd mkt # you get error message, if cd mkt file doesn't exist, but it creates and cd to mkt

pwd

cd 

rm -r mkt

cd mkt 2>/dev/null || mkdir mkt && cd mkt  # you doesn't get error message because error redirected to /dev/null

pwd

cd 

cd mkt 2>/dev/null || mkdir mkt && cd mkt # you got cd error message, 
becuase each command independent so in background

cd mkt 2>/dev/null || mkdir mkt  # cd mkt success so skip the mkdir mkt, than
cd mkt # error because no directory

solution: grouping the commands

cd 

rm -r mkt

cd mkt 2>/dev/null || { mkdir mkt && cd mkt; }

pwd

cd

cd mkt 2>/dev/null || mkdir mkt && cd mkt

pwd

cd mkt 2>/dev/null || mkdir mkt && cd mkt

pwd



------------ shell scripts ---------------

sudo yum install -y vim-enhanced

we can add abbrevations into ~/.vimrc  # shortcuts
ex:
	echo 'abbr _sh #!/bin/bash' >> ~/.vimrc


how to know the type of file?

	file <filename.extension>
	

- Script interpreter/shell interpreter


PATH Environment variable
--------------------------
we can run the script irrespective of file path. To do this we need add the file path to PATH environmental variable.

example
touch my.sh

mkdir bin && mv my.sh bin/

chmod -v +x my.sh


which my.sh

 
Special variables
-----------------

$$ - current PID
$? - exit status of previous command
!$ - Last argument  (useful in scripts)
$0 - program names
$1 - first argument
$# - arguments count
$* - all arguments as a string
$@ - all arguments as an array


------------- Automating the user creation process -----------------

Question:

Create a script taking an argument for the username, the script should not proceed if the name is not supplied.
Use conditional statements to ensure we only try to create the account if it does not already exit.
The password will be collected during script execution using the read command.
For confirmation, the new user account deatils are printed to the screem.


----------------Functions and loops------




