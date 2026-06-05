
> [!NOTE] Module 8 - Assignment
> **Problem Statement:**
> You work for xyz organization. Your job work is to manage Linux-based servers.
> 
> You have been asked to:
> 1. Launch Firefox 3 times, find their PID and use the kill command to close them
> 2. Monitor incoming logs files to a folder using inotify and output them to a file
> 

### Task 1:

Launched Firefox 3 times:  
![[Pasted image 20230820123022.png]]

After launching Firefox, I used the `pgrep` command to observe the multiple processes that were spawned:
```bash
pgrep -f firefox
```
![[Pasted image 20230820133423.png]]

Now we terminate these processes by piping these PIDs to the `kill -9` command using `xargs` and confirm with `pgrep -f firefox`
```bash
pgrep -f firefox | xargs -r kill -9
pgrep -f firefox #confirming termination
```
![[Pasted image 20230820133556.png]]

### Task 2:

First I need to install inotify utility

```
sudo apt update -y
sudo apt install inotify-tools -y 
```

I'll create a folder to monitor called `log_folder` in `~/Desktop`
```
mkdir log_folder
```

In order to monitor this folder and output the monitoring to a separate file I'll create script `log_monitor.sh` with the following code:

```bash
#!/bin/bash

# Directory to watch
WATCH_DIR="/home/hector/Desktop/log_folder"

# Output file
OUTPUT_FILE="/home/hector/Desktop/inotifywait_log"

# Remove the output file if it already exists
if [ -e "$OUTPUT_FILE" ]; then
  rm "$OUTPUT_FILE"
fi

# Create the output file
touch "$OUTPUT_FILE"

# Start the monitoring
inotifywait -m -e close_write --format '%w%f' "$WATCH_DIR" | while read -r NEW_FILE; do
  echo "New file detected: $NEW_FILE"
  echo "Appending contents to $OUTPUT_FILE"
  echo "$NEW_FILE" >> "$OUTPUT_FILE"
done
```
The script uses `inotifywait` with `-e close_write` which is triggered when a file is created or written to. Furthermore, the script's output will be recorded in file `inotifywait_log` in `~/Desktop`

Need to make the script `log_monitor.sh` executable
```bash
chmod +x log_monitor.sh
```

I'll execute the script to start monitoring
```bash
./log_monitoring.sh
```
![[Pasted image 20230820145952.png]]

To test the monitoring I'll start creating files inside `log_folder`  
![[Pasted image 20230820150147.png]]

We now see our running script has some console output  
![[Pasted image 20230820150231.png]]

Now we check the output file generate by the script `inotifywait_log` an see the files created  
![[Pasted image 20230820150400.png]]

