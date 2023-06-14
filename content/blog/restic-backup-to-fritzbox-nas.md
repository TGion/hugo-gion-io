---

title: "Backup to a FRITZ!Box NAS with restic and rclone"  
date: 2022-12-29T08:00:00+02:00  

draft: false  

description: "Setup restic and rclone to backup a FreeBSD based VPS to storage attached to a FRITZ!Box. Create a backup script and run continuous backups with cron."  
keywords: [backup, restic, rclone, nas, freebsd]  

tags: [bsd]  
toc: false

---

Data loss sucks! We've been all there.
So one of the first things to setup on a fresh server is some sort of backup solution. The choice of software and strategies are huge and I am by no means an expert on this topic (or any). 

Nonetheless, I will show you my setup using a sweet backup software called [restic](https://restic.net/). It's fast, secure and of course, open source.

Since I like to have my backups stored at home (also I am a cheap bastard), I will use [rclone](https://rclone.org/) to save the backup directly to my harddrive connected to my FRITZ!Box. This gives me the benefit of cheap and physically available backup storage.

## Goals

After the completion of this guide, you will be able to:

1. Configure rclone to access your storage connected to your FRITZ!Box.
2. Setup restic to backup your desired files to the rclone repository.
3. Create a backup script and cronjob to periodically create incremental backups.

## Prerequisites

Make sure you have your [FRITZ\!Box](https://en.avm.de/service/knowledge-base/dok/FRITZ-Box-7340-int/1001_Accessing-the-FRITZ-Box-over-the-internet/) set up correctly:
- Able to access it via a dynamic hostname (e.g. myFRITZ!)
- Allow an user to access the storage via FTPS 

## Software Installation

To install restic and rclone you can either use the [FreeBSD port collection](https://www.freebsd.org/ports/) or the [pkg](https://docs.freebsd.org/en/books/handbook/ports/#pkgng-intro) manager. I prefer pkg and therefore will omit the installation via the ports collection.

As root, update the pkg repository and install restic and rclone.

```bash
pkg update
pkg install restic rclone
```
Done!

## Rclone Configuration

Rclone is able to synchronize files to many different [providers](https://rclone.org/#providers) and has a lot of very cool [features](https://rclone.org/#features). It is quite useful in every scenario where you have to manage files in and outside of cloud storages.

To create a new configuration for a data storage, you have to run (as root):

```bash
rclone config
```

You will be guided through an interactive configuration wizard where you need to enter your account settings for your FRITZ!Box.

```bash
[RCLONE_CONFIG_NAME]
type = ftp
host = xyz.myfritz.net
user = myuser
port = 48656
pass = mypasswd
```
You also need to set options to make FTP/S work with the FRITZ!Box:

```bash
concurrency = 1          // Only one concurrent FTP connection for more stability (advanced settings)
tls = false              // Implicit TLS does not work with FRITZ!Box
explicit_tls = true      // Explicit TLS enabled
```

You should test the connection with the following command, where ```REMOTE_FOLDER``` is a valid folder on your harddrive connected to your FRITZ!Box. If everything is set up correcty you will get back the directory listing of ```REMOTE_FOLDER```:

```bash
rclone ls RCLONE_CONFIG_NAME:/REMOTE_FOLDER
```

## Restic configuration 

In order to backup any files with restic, we need a repository. It's basically a set of files and directories where snapshots are saved. No databases or config files needed. Just simply elegant.

The repository will be created on your previously configured rclone storage ```RCLONE_CONFIG_NAME``` in the folder ```restic-repo```. The folder should be created manually in advance.

Due to limitations of concurrent connections to the FRITZ!Box and restic being unable to read the settings from our rclone setup, we need to add ```-o rclone.connections=1``` to avoid [connection locks](https://github.com/rclone/rclone/issues/4694) when accessing the FRITZ!Box.

```bash
restic init --repo rclone:RCLONE_CONFIG_NAME:/restic-repo -o rclone.connections=1
```

And the confirmation after setting your password (twice to confirm):

```bash
created restic repository 085b3c76b9 at rclone:RCLONE_CONFIG_NAME:/restic-repo 
Please note that knowledge of your password is required to access the repository.
Losing your password means that your data is irrecoverably lost.
```

Now you have your repository initialized and can start backing up files to it:

```bash
restic --repo rclone:RCLONE_CONFIG_NAME:/restic-repo backup /etc -o rclone.connections=1
```
Restic will open the repository ```rclone:RCLONE_CONFIG_NAME:/restic-repo``` for which it will demand your password and save all files and subfolders from ```/etc``` to it. If you run the command a second time, you will notice that only the changes will be added (increment) and therefore almost little to no data.

Congratulations! You made A backup of A folder.
Let's dig deeper....

### Password file

In order to use restic in your soon to have backup script, you can't put in your password manually to open the repository. 

Create a new file containing your password as user root. I suggest you put the file in ```/usr/local/etc``` as best practice. Make sure to set the appropriate file mode (root only read- and write access):

```bash
cd /usr/local/etc
echo 'SUPER_SECRET_PASSWORD' > restic-repo.passwd
chmod 600 restic-repo.passwd
```

To make restic use a password file you need to extend our previous command like with the option ```-p PASSWD_FILE```:

```bash
restic -p /usr/local/etc/etc/restic-repo.passwd --repo rclone:RCLONE_CONFIG_NAME:/restic-repo backup /etc -o rclone.connections=1
```

### Setting environments

You probably noticed that our command gets tedious long already. To make our commands (and soon backup script) better readable, we can use environment variables which restic will read if set:

```bash
export RESTIC_PASSWORD_FILE=/usr/local/etc/restic-repo.passwd
export RESTIC_REPOSITORY=rclone:RCLONE_CONFIG_NAME:/restic-repo
```

This will shorten our previous command to ```restic backup /etc -o rclone.connections=1```. 

Sweet!

### Good practices

**Include Files**

Maybe you dont want to edit your backup script every time you want to add additional folders to your backup. Or you want to exclude file contained in your included folders. Here are some approaches on how to implement that:

Create a file called ```restic-repo.include``` in ```/usr/local/etc``` and change its file mode. You know the drill!

```bash
cd /usr/local/etc
touch restic-repo.include
chmod 600 restic-repo.include 
```

Use your [favorite editor](https://micro-editor.github.io/) to add folders to ```restic-repo.include``` to include them in your backup.
Previously we wanted to backup ```/etc```, so our ```restic-repo.include``` should contain the folder ```/etc```.

This will backup ```/etc``` and every subfolder. All you have to do now is to change the previous restic command to use your ```restic-repo.include```:

```bash
restic backup $(cat /usr/local/etc/restic-repo.include) -o rclone.connections=1
```

**Exclude Files**

The process is pretty similar if you want to exclude files. In this simple example we only exclude a subfolder from a included folder. If you want a more complex exclude logic, you should check out the [restic documentation](https://restic.readthedocs.io/en/latest/040_backup.html) on that topic.

Create a file called ```restic-repo.exclude``` in ```/usr/local/etc``` and change its file mode.

```bash
cd /usr/local/etc
touch restic-repo.exclude
chmod 600 restic-repo.exclude 
```

To exclude ```/etc/rc.d``` from our backup we add the folder to ```restic-repo.exclude```.

Finally you have to extend your command:

```bash
restic backup $(cat /usr/local/etc/restic-repo.include) --exclude-file=/usr/local/etc/restic-repo.exclude -o rclone.connections=1
```

**Show diff of last backup**

To show the difference between two snapshots, we can use ```restic diff```. This will give us all the files which have been added, removed or changed between two selected snapshots. To get the last two snapshots we can use ```restic snapshots --compact``` and some magic.

```bash
restic diff $(restic snapshots --compact | tail -n4 | head -n2 | cut -d' ' -f1)
```
Or logs will now contain a list of files which have been added in the latest backup.

## Backup script

Now you have everything set up to combine it into one backup script.

As root create a new file ```/usr/local/sbin/restic-backup.sh``` and set the appropriate file permissions. Since the script should be executed we have to set the +x bit (chmod 700):

```bash
cd /usr/local/sbin
touch restic-backup.sh
chmod 700 restic-backup.sh
```

Use your [favorite editor](https://micro-editor.github.io/) ```restic-backup.sh``` to create your script:

```bash
#/bin/sh

# Init
export RESTIC_PASSWORD_FILE=/usr/local/etc/restic-repo.passwd
export RESTIC_REPOSITORY=rclone:RCLONE_CONFIG_NAME:/restic-repo

echo "######################################################"
echo "Starting backup @ $(date)"
echo "######################################################"

# Run programs before your backup is done, e.g.:
# - Enable Mainenance mode of Nextcloud
# - Dump PostgreSQL Database to include in backup
# - Sleep

# Backup files with restic (rclone)
restic backup $(cat /usr/local/etc/restic-repo.include) --exclude-file=/usr/local/etc/restic-repo.include -o rclone.connections=1

# Summary of backup
restic diff $(restic snapshots --compact | tail -n4 | head -n2 | cut -d' ' -f1)

# Run programs AFTER your backup is done, e.g.:
# - Disable Mainenance mode of Nextcloud
# - Delete temporary database dump 
# - Send mail to admin

```

## Crontab

You can create a new crontab entry (as root) to run your script whenever you want (daily, weekly, etc.):

```bash
crontab -e
```
The following example entry runs the backup script ```restic-backup.sh``` every night @ 04:00 and appends the output to ```/var/log/restic.log```:

```bash
0 4 * * * 	restic-backup.sh >> /var/log/restic.log
```

## Conclusion and ideas for improvement

I hope this simple example helps you setup your own backup strategy with restic and rclone so you can save your backups to your harddrive connected to your FRITZ!Box. 

There are quite a few topics I haven't covered or could be improved:
- Access FRITZ!Box FTP via Wireguard and therefore restrict public access to FTP/S Server
- Create newsyslog log file rotation
- Send summary of backups via mail to yourself

Still, I hope you enjoyed this guide and found some input useful.





