```bash
$ while IFS= read -r var; do
while> echo "Hello $var"
while> done < text.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# automates bash commands with input from a .txt file
```


```bash
mv `ls -I directory -I venv` directory
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# moves everything in the current directory into another directory
```


```bash
awk -i inplace '!seen[$0]++' text.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# removes duplicates from a file
```


```bash
find / -name "text.txt" 2&>0 -exec vim {} \;
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# finds a file and instantly edits it with vim
```


```bash
grep -r "password" .
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# recursively greps for the keyword starting from the current directory
```


```bash
sudo systemctl start/stop gdm3
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# starts/stops the GUI (if gdm3 is used). A shell can be started with a turned off GUI using CTRL+ALT+F2
```


```bash
tmux
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
Ctrl+B D # — Detach from the current session.
Ctrl+B % # — Split the window into two panes horizontally.
Ctrl+B "" # — Split the window into two panes vertically.
Ctrl+B Arrow Key (Left, Right, Up, Down) # — Move between panes.
Ctrl+B X # — Close pane.
Ctrl+B C # — Create a new window.
Ctrl+B N or P # — Move to the next or previous window.
Ctrl+B 0 (1,2...) # — Move to a specific window by number.
Ctrl+B : # — Enter the command line to type commands. Tab completion is available.
Ctrl+B ? # — View all keybindings. Press Q to exit.
Ctrl+B W # — Open a panel to navigate across windows in multiple sessions.
```


```bash
scp <file to upload> <username>@<hostname>:<destination path>
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# uploads a file using ssh
```

```
scp <username>@<hostname>:<file to receive> <destination path>
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# downloads a file using ssh
```


```bash
python3 -m venv venv
source ./venv/bin/activate
pip3 install impacket
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# creates a virtual environment for python, which can be removed quickly. Thereby not touching system python libraries
```


```bash
ssh-keygen -R <ip>
vim ~/.ssh/known_hosts # remove conflicting host
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# clears `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`
```


```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="apparmor=0"/' /etc/default/grub && sudo update-grub && sudo reboot
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# stops AppArmor
```


```bash
/<search>
n
n
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# search for keywords in vim, similar to CTRL+F, stepping to other points with n
```


```bash
awk '{line=substr($0,13); split(line,a," "); print a[1]}' file.txt > output.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# removes the first 12 characters, and removes everything after a space
# good to extract IPs from a nxc scan!
```


```bash
sort -n filename.txt > sorted.txt
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# sorts text files
```


```bash
tail -f /var/log/apache2/error.log
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# continously monitor a logfile
```


```bash
ps -eo user,comm | grep -i apache2
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# shows who has started that process
```


```bash
tar -czf archive.tgz file1 dir1 file2 dir2 ...
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# creates .tgz archive
```

```
tar -xvzf archive.tgz
```
<div class="annotate" markdown> (1) </div>

1. 
```bash
# unzips .tgz archive
```
