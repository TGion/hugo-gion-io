---

title: "Automate tasks with cron, periodic and at"  
date: 2023-05-29T08:00:00+02:00  

draft: false  

description: "Configure automated system and user tasks with integrated FreeBSD system tools cron, periodic and create new tasks. Deliver reports via mail with dma to the system user."  
keywords: [freebsd, automation, cron, periodic.conf]  

tags: [bsd]  
toc: true

---

Running and maintaining a server often requires regular jobs, scripts, command to be run in order to keep the system running smooth. Think of log rotations, disk cleanups, flushing mail queues, etc.

FreeBSD not only offers a great set of tools with [cron](https://man.freebsd.org/cgi/man.cgi?cron), [periodic](https://man.freebsd.org/cgi/man.cgi?periodic(8)) and [at](https://man.freebsd.org/cgi/man.cgi?query=at&sektion=1&manpath=FreeBSD+13.2-RELEASE+and+Ports), but also those tools are tightly integrated into the base system. Which is always good news for security, compatibility and performance.

## Goals

After the completion of this guide, you will be able to:

1. Create individual automation tasks for different users.
2. Configure and extend the Unix (FreeBSD) periodic system as well as enable mail delivered reports.
3. Use `at` to independently run time-consuming tasks once.

## Prerequisites

Nothing special. I assume you have some basic knowledge about Unix systems and how to edit files ;)

## Crontab

I used to run every automated task through (different) user's crontab files. Although I have learned to use more elegant ways for automation, the user's crontab remains a valid configuration for repetative jobs, if:

1. It is a individual task for a specific user and not useful for the system or other users.
2. The task need's to run several times a day.

### Creating crontab's

In order to setup automation via crontab, the user's crontab file must be edited to add the command or script to the file. This also includes the time at which the task should be run regularly.

Every user on the system has a crontab file (including root). Depending on your role on the server (user / admin), security concerns or permissions, you can either setup cron's for your own user, a different user or root itself.

To edit your own crontab (can also be root if logged in as), you have to run:

```
crontab -e
```

If you want to edit another user's file, you simply have to add `-u [user]`. Every job in the user's crontab will then be executed as the designated user:

```
crontab -e -u joe
```

### Structure of crontab

Every entry of the crontab file consists of the command or and the time at which it should run (we will discuss environment settings later). I urge you to read the [manpage](https://man.freebsd.org/cgi/man.cgi?query=crontab&sektion=5&manpath=FreeBSD+13.2-RELEASE+and+Ports) for more details, but the basic usage is as follows:

```
### minute  hour	mday	month	wday	command

### Flush dma mail queue every 30 minutes
    */30    *       *       *       * 	    /usr/libexec/dma -q

### Every 3 hours update PF's firewall rules 
    0       */3     *       *       *	    pf-badhost -O freebsd
```

### Setting environment in crontab

Crontab's are run with `/bin/bash` in a "clean" environment. This means, it may be necessary for you to take care of environment settings (e.g.):

```bash
# Override default shell
SHELL=/bin/sh

# Add more directory to the PATH
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin

# Mail any output to the user joe, even if its not his crontab
MAILTO=joe
```

This is done in the beginning of the crontab file **before** any command is run.

### Cons of using crontab

There are two downsides when using (only) user's crontab file:

1. Having many jobs can make it difficult to keep track of them. Especially if different crontabs are involved. You don't want to run multiple jobs at the same time. Also you don't want to run the same job in different crontab's.
2. You have to take care for delivering reports in every job, or at least in every crontab. Also you will get plenty of mails this way.

To adress those issue, we can move many of our jobs to the periodic system and configure the system defaults to our needs.

## Periodic

The periodic system consists of a binary `periodic` and a directory structure `daily/weekly/monthly/security` for system and user scripts. They are run periodicaly from the system's crontab.

### System periodics

FreeBSD comes with several pre-configured periodical jobs for regular system maintenance. They are located in:

```
/etc/periodic/daily
/etc/periodic/weekly
/etc/periodic/monthly
/etc/periodic/security
```

The default settings when they are run can be configured in `/etc/crontab`:

```
### minute	hour	mday	month	wday	who	    command
### Perform daily/weekly/monthly maintenance.
    1	    2	    *	    *	    *	    root	periodic daily
    15	    4	    *	    *	    6   	root	periodic weekly
    30	    5	    1	    *	    *	    root	periodic monthly
```

To configure how and what jobs should be run, you need to edit `/etc/periodic.conf`, which is empty by default. I suggest you copy the *default* one from:

```
cp /etc/defaults/periodic.conf /etc/periodic.conf
```

and edit it for your needs. Entries which should run with their default settings can be safely removed from `/etc/periodic.conf`.

### Create new periodics

User and 3rd party periodics are (and should be) located in:

```
/usr/local/etc/periodic/daily
/usr/local/etc/periodic/weekly
/usr/local/etc/periodic/monthly
/usr/local/etc/periodic/security
```

Scripts in those directories are run in order of their file name. To prioritize them, a numeric prefix is added, e.g.: `100.clean-disks`. After you have decided when to run your script, you can create a new file (or copy an existing job) in the appropiate directory, e.g. `/usr/local/etc/periodic/daily`:

We will create a new periodic job to check for updates of our installed packages and update them accordingly. Create a file `/usr/local/etc/periodic/daily/412.pkg-updates`:

```bash
#!/bin/sh -
#
# $FreeBSD$
#

# If there is a global system configuration file, suck it in.
#
if [ -r /etc/defaults/periodic.conf ]
then
    . /etc/defaults/periodic.conf
    source_periodic_confs
fi
```

For every periodic this is the default which can be used every time. It defines `/bin/sh` to run the script and reads the system configuration defaults from `/etc/defaults/periodic.conf` if it exists:

Now we add the ability to enable / disable our script through `/etc/periodic.conf`:

```bash
case "$daily_pkg_updates_enable" in
    [Yy][Ee][Ss])

    # Our commands will be run here!

    ;;
    
    *)  rc=0;;
esac
```

So the complete script will be:

```bash
#!/bin/sh -
#
# $FreeBSD$
#

# If there is a global system configuration file, suck it in.
#
if [ -r /etc/defaults/periodic.conf ]
then
    . /etc/defaults/periodic.conf
    source_periodic_confs
fi

case "$daily_pkg_updates_enable" in
    [Yy][Ee][Ss])

    echo ""
    pkg update

    # echo "Checking for pkg updates:"
    pkg version -vURL=
    
    # echo "Upgrading pkgs:"
    pkg upgrade -y

    ;;
    
    *)  rc=0;;
esac
```

Make sure your new file can be executed:

```
chmod +x /usr/local/etc/periodic/daily/412.pkg-updates
```

Since we added the ability to control our periodic in `/etc/periodic.conf` we need to enable it:

```
echo 'daily_pkg_updates_enable="YES"' >> /etc/periodic.conf
```

Altough you can name you variable the way you want, the name should describe when and what you run with your periodic. 

### Mail reports

The periodic system by default sends a summary report about every periodic script it has run. This will also include custom created periodic's as long as they are managed with `/etc/periodic.conf`.

Those mails are locally sent to the user `root`. If you want to them to be delivered to an external mail address, you have to edit `/etc/aliases` and add an mail adress to root:


root:   mail@address.tld


If you don't have a mail server running you can you use [dma - a small mail transport agent](https://github.com/corecode/dma). I will show you in a another post how to set it up. Or you use another awesome blog post about [Setting up dma with FreeBSD](https://herrbischoff.com/2021/10/freebsd-13-simple-outgoing-email-with-dma/).

## at

If you have time intensive jobs to run, you may not want your periodic system to wait for them. E.g. you want your daily reports as soon as possible to check for failures.

To de-couple intensive jobs FreeBSD offers us `at` to queue jobs for delayed execution. Since those jobs are again picked up by another cron jobs (consistency anyone?), it will run independent from our periodic system.

By default at expects it's jobs to be entered interactive. Not much use for scripts, so you can create a job file to list the commands you want to run. In my example I run [synth](https://github.com/jrmarino/synth) to build new port updates. Here is `synth-jobs`:

```bash
# Here comes a list of commands
# Needed for synth to run in scripts
export TERM=dumb

echo "Build packages and upgrade system:"
/usr/local/bin/synth upgrade-system
```

Now you can tell `at` to run you job file and mail the results to the user who fired the job:

```
at -m -f synth-jobs now
```

The job will be queued for immediate execution. But since [atrun](https://man.freebsd.org/cgi/man.cgi?query=atrun&sektion=8) which executes `at` jobs is managed by cron, it will be only picked up every 5 minutes (default on FreeBSD). See `/etc/cron.d/at` for more information or to shorten the intervall.


## Conclusion

FreeBSD (and it's brothers and sisters) offer a great set of tools as part of their base system to automate tasks and report the results by mail. It offers great flexibility and control with almost no overhead.

More advanced and future topics could be:

- Generalize periodics by extending the use of parameters in `/etc/periodic.conf`
- Add logic for failure and error recovery in case tasks fail
- Event based triggers to run `at` jobs

Have fun with the automation of your tasks and take care!
