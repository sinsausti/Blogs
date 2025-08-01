# Mastering Logrotate: Keep Your Server Logs Under Control

Hey there, fellow tech enthusiasts! 👋 

If you've ever managed a server for more than a few weeks, you've probably noticed something: log files have an annoying habit of growing... and growing... and growing until they've eaten up all your disk space. Sound familiar? That's where our hero **logrotate** comes to the rescue!

## What is Logrotate?

Think of logrotate as your server's personal housekeeper. It's a nifty utility that automatically manages your log files by rotating, compressing, and cleaning them up on a schedule you define. No more manually deleting old logs or panicking when your disk space hits 100%!

## Why Should You Care?

Let's be honest – nobody wants to wake up to a server that's crashed because the logs filled up the entire disk. Logrotate prevents this nightmare scenario by:

- **Rotating logs automatically** (moving old logs to archived versions)
- **Compressing old files** to save precious disk space
- **Removing ancient logs** that you probably don't need anyway
- **Maintaining system performance** by keeping active log files manageable

## A Real-World Example

Let's break down a practical logrotate configuration that handles Apache logs across multiple virtual hosts:

```bash
/var/log/apache2/*.log /var/log/apache2/sms.br.binbit.com/*.log /var/log/apache2/platform/*.log /var/log/apache2/tigo.py.binbit.com/*.log /var/log/apache2/entelcl.sms.binbit.com/*.log {
    daily
    missingok
    rotate 26
    compress
    notifempty
    create 644 platform platform
    sharedscripts
    dateext
    postrotate
        /etc/init.d/apache2 reload > /dev/null
        camino=`dirname $1`
        /bin/cp $camino/*.gz /logs/platform/Anchorhead/apache2/
        echo "$camino $1" >> /home/sinsausti/logrotate.log
    endscript
}
```

### Breaking It Down

Let's decode what each directive does:

**File Patterns**: The configuration targets multiple Apache log directories, including main logs and virtual host-specific logs.

**Key Directives Explained**:
- `daily` – Rotate logs every day (you could also use `weekly` or `monthly`)
- `missingok` – Don't freak out if a log file is missing
- `rotate 26` – Keep 26 old versions (roughly 6 months of daily logs)
- `compress` – Gzip the old logs to save space
- `notifempty` – Skip rotation if the log file is empty
- `create 644 platform platform` – Create new log files with specific permissions and ownership
- `sharedscripts` – Run the postrotate script only once, even if multiple files match
- `dateext` – Add date extensions to rotated files (like `.20241231`)

**The postrotate Script**: After rotation, this configuration:
1. Reloads Apache to recognize the new log files
2. Copies compressed logs to a backup location
3. Logs the rotation activity for auditing

## Common Gotchas and Tips

### The "No Such File" Error
You might encounter errors like:
```
/bin/cp: cannot stat `/var/log/apache2/*.gz': No such file or directory
```

This usually happens when the script tries to copy compressed files that don't exist yet (like on the first run). You can handle this gracefully with:

```bash
postrotate
    if ls $camino/*.gz 1> /dev/null 2>&1; then
        /bin/cp $camino/*.gz /logs/platform/Anchorhead/apache2/
    fi
endscript
```

### Pro Tips for Success

1. **Test your configuration** with `logrotate -d /etc/logrotate.conf` (dry run mode)
2. **Check permissions** – make sure logrotate can read/write to all specified directories
3. **Monitor disk space** even with logrotate running
4. **Consider using `copytruncate`** for applications that keep log files open
5. **Use `olddir`** to move old logs to a separate directory if needed

## Setting Up Your Own Logrotate

Most Linux distributions come with logrotate pre-installed. Your configuration files typically live in:
- Main config: `/etc/logrotate.conf`
- Service-specific configs: `/etc/logrotate.d/`

For Apache specifically, you might create `/etc/logrotate.d/apache2` with your custom configuration.

## Wrapping Up

Logrotate might not be the most exciting tool in your sysadmin toolkit, but it's definitely one of the most important. Set it up once, and it'll quietly keep your servers happy and your disk space under control.

Remember: a well-configured logrotate setup is like a good backup strategy – you'll only appreciate it when you need it most!

Have you had any interesting experiences with logrotate? Found any clever configurations that saved the day? I'd love to hear about them in the comments below!

---

*Happy logging, and may your disks never fill up! 🚀*