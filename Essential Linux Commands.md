# Essential Linux Commands Every System Admin Should Know

## Welcome to Your Linux Command Toolkit!

Being a Linux system administrator is like being a digital Swiss Army knife – you need the right tool for every situation. Over the years, I've collected some incredibly useful commands that have saved my bacon more times than I can count. Today, I'm sharing this treasure trove with you!

Whether you're troubleshooting a server at 3 AM, cleaning up disk space, or hunting down that mysterious process eating all your CPU, these commands will become your best friends.

## File Search and Text Operations

### Finding Files with Specific Content

Need to hunt down files containing specific text? This one's a lifesaver:

```bash
# Search for text in current directory and subdirectories
grep -lir "some text" *

# More advanced search with find and grep
find . ! -name . -prune -name '*' -print0 | xargs -0 grep "texto" /dev/null
```

### File Management by Age

Clean up old files automatically:

```bash
# Delete files older than 7 days in current directory
rm -rf `find -maxdepth 1 -mindepth 1 -mtime +7`

# Find and delete files older than one year
find <directory path> -mtime +365 -and -not -type d -delete

# List files by modification time (great for finding recent changes)
find /home/user -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort
```

### Finding Large Files

Running out of disk space? Find the culprits:

```bash
# Find files larger than 10MB
find / -type f -size +10M

# Top 10 largest directories
du -sb * | sort -nr | head | awk '{print $2}' | xargs du -sh

# Find directories with unusual sizes
find . -type d -exec ls -ld {} \; | awk '{ if ( $5 > 4096 ) print "Size: " $5/1024/1024 " Mb " $9}' > ~/large_dirs.txt
```

## Network Diagnostics and Monitoring

### Port and Connection Analysis

```bash
# Find what's listening on port 80
netstat -ntlp | grep 80 | awk '{print $7}' | cut -d/ -f1

# Show established connections
lsof -i | grep -i estab

# Find listening ports by PID
lsof -nP +p 24073 | grep -i listen | awk '{print $1,$2,$7,$8,$9}'

# Graph connections per host
netstat -an | grep ESTABLISHED | awk '{print $5}' | awk -F: '{print $1}' | sort | uniq -c | awk '{ printf("%s\t%s\t",$2,$1) ; for (i = 0; i < $1; i++) {printf("*")}; print "" }'
```

### Network Traffic Monitoring

```bash
# Monitor network traffic (excluding your SSH session)
tcpdump -i eth1 -s 1500 port not 22

# Skip multiple ports
tcpdump -i eth1 -s 1500 port not 22 and port not 53

# Monitor specific host
tcpdump -i eth1 port not 22 and host 1.2.3.4
```

## Process Management and System Resources

### Advanced Process Analysis

```bash
# Processes sorted by CPU usage
ps -e -o pcpu,cpu,nice,state,cputime,args --sort pcpu | sed '/^ 0.0 /d'

# Processes sorted by memory usage
ps -e -orss=,args= | sort -b -k1,1n | pr -TW$COLUMNS

# CPU and memory usage by user
ps -eo user,pcpu,pmem | tail -n +2 | awk '{num[$1]++; cpu[$1] += $2; mem[$1] += $3} END{printf("NPROC\tUSER\tCPU\tMEM\n"); for (user in cpu) printf("%d\t%s\t%.2f%%\t%.2f%%\n",num[user], user, cpu[user], mem[user]) }'

# Memory consumption by specific processes (adjust grep pattern)
ps -e -orss=,args= | sort -b -k1,1n | pr -TW$COLUMNS | egrep "(php|python|apache|postgre)" | grep -v grep | awk '{ mem += $1 } END { print mem/1024"MB" }'
```

### System Resource Monitoring

```bash
# Count total open files
lsof | wc -l

# List files opened by a specific PID
lsof -p 15857

# Check which program owns a port
lsof -i tcp:80

# System uptime and load
uptime

# Memory statistics
free -h
```

## Useful System Administration Tricks

### Quick System Info

```bash
# Complete system information
uname -a

# Current hostname
hostname

# List all users currently logged in
w | egrep -v '(load|FROM)' | awk '{print $2}' | sed 's/^/tty/'

# Send a message to all terminals (just for fun!)
w | egrep -v '(load|FROM)' | awk '{print $2}' | sed 's/^/tty/' | awk '{print "echo \"The Matrix has you...\" >> /dev/" $1}' | bash
```

### MySQL Quick Commands

```bash
# Show MySQL process IDs
mysql -s -e "show processlist" | awk '{print $1}'

# Monitor MySQL processes continuously
#!/bin/bash
while [ 1 ]; do
    mysql -N -u root -ppassword -e 'show processlist' | grep -v 'show processlist'
    sleep 2
done
```

### Smart Daemon Management

```bash
# Kill daemon by name (not PID)
kill_daemon() { 
    echo "Daemon?"; 
    read dm; 
    kill -15 $(netstat -atulpe | grep $dm | cut -d '/' -f1 | awk '{print $9}') 
}
alias kd='kill_daemon'
```

## File and Directory Operations

### Advanced File Operations

```bash
# Change MAC address
ifconfig eth0 hw ether 00:11:22:33:44:55

# Remove backup files in home directory
find ~user/ -name "*~" -exec rm {} \;

# Convert DOS line endings to Unix
perl -pi -e 's/\r\n/\n/g' filename

# List installed fonts
fc-list | cut -d ':' -f 1 | sort -u

# Check symlink status
symlinks -r $(pwd)
```

### Archive and Transfer

```bash
# Mail files as attachment
tar cvzf - data1 data2 | uuencode data.tar.gz | mail -s 'data' you@host.fr

# Resume interrupted scp
rsync --partial --progress --rsh=ssh $file_source $user@$host:$destination_file

# Send your IP by email
ifconfig en1 | awk '/inet / {print $2}' | mail -s "hello world" email@email.com
```

## Log Analysis and Monitoring

### Apache Log Analysis

```bash
# Analyze IP frequency in Apache logs
cat /var/log/apache2/access.log | awk '{ print $5 }' | cut -d ':' -f1 | sort | uniq -c | sort | tail

# Monitor log files in real-time
tail -f /var/log/messages

# Show last 15 lines and keep monitoring
tail -f --lines 15 /var/log/messages
```

### Command History Analysis

```bash
# See most used commands
history | awk '{print $2}' | awk 'BEGIN {FS="|"} {print $1}' | sort | uniq -c | sort -r
```

## Database Operations

### PostgreSQL Quick Commands

```bash
# List all PostgreSQL databases (excluding templates)
psql -U postgres -lAt | gawk -F\| '$1 !~ /^template/ && $1 !~ /^postgres/ && NF > 1 {print $1}'
```

## Cleanup and Maintenance Scripts

### Automated Cleanup

```bash
#!/bin/bash
# Clean old archive files
find /u1/database/prod/arch -type f -mtime +3 -exec rm {} \;

# Clean compressed files older than 30 days
find /path/dir -name "*.bz2" -type f -mtime +30 -delete
```

### System Maintenance

```bash
# Force filesystem check on reboot
shutdown -rF now

# Rescan for new disks (VMware)
echo "- - -" > /sys/class/scsi_host/host0/scan
```

## Pro Tips for Daily Use

### Terminal Productivity

```bash
# Terminal redirection (share your terminal with another user)
script /dev/null | tee /dev/pts/3

# Check XML file validity
curl -s 'http://example.com/file.xml' > file.xml
xmlwf file.xml
```

### File Split and Processing

```bash
# Split large files into 2MB chunks
split -b 2m largefile largefile_

# Process tree view
pstree
```

## System Information Commands

### Hardware and System Stats

```bash
# Kernel messages
dmesg

# Module management
lsmod                    # List loaded modules
modprobe module_name     # Load a module
rmmod module_name        # Remove a module

# System limits
ulimit -a

# Kernel parameters
sysctl -a
```

## Storage and Filesystem

### Disk Usage Analysis

```bash
# Disk usage summary
df -h

# Directory size analysis
du -sh /*

# Find 20 biggest directories
du -xk | sort -n | tail -20
```

This command collection has been my go-to reference for years. Bookmark this page, practice these commands in a safe environment, and soon you'll be wielding Linux like a pro!

Remember: with great power comes great responsibility. Always test commands on non-production systems first, and keep backups of important data. Happy administering! 🚀

*Pro tip: Create aliases for your most-used commands to save time and reduce typos!*