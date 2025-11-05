# Bash-Scripting-TASK

# 🖥️ System Statistics Monitoring Project

This project collects **CPU**, **Memory**, and **Disk usage** data from the system every hour, calculates averages, and displays them on a local **Apache web page**.

---

## 📂 Project Structure

```
/opt/sysstats/
│
├── collect.sh # collects CPU, MEM, DISK data every hour
├── avg.sh # calculates averages and updates HTML pages
├── raw/ # stores collected data (timestamped .txt files)
├── avg/ # stores calculated average results
└── logs/ # stores log files for both scripts
```
```sql
Web files are stored in:
/var/www/html/sysstats/
│
├── index.html
├── cpu.html
├── mem.html
└── disk.html
```
## 🧩 Section 1 – Collecting System Statistics
**Question:**  
Create a shell script that collects CPU, Memory, and Disk usage every hour and saves the results in `/opt/sysstats/raw/` with timestamps.

---

**Answer:**

### 📜 Script: `collect.sh`
```bash
#!/bin/bash

RAW=/opt/sysstats/raw
LOG=/opt/sysstats/logs/collect.log


#timestamp
TIME=$(date +"%Y-%m-%d-%H-%M")


#CPU Usage
CPU=$(vmstat 1 2 | tail -1 | awk '{print 100 - $15}')


#Memory Usage
MEM=$(free -m | awk '/Mem:/ {printf "%.2f", ($3/$2)*100}')

#Disk Usage
DISK=$(df -m / | awk '{print $5}' | sed 's/%//' | tail -1)

#Results saved in files:
echo "$CPU" > "$RAW/cpu_$TIME.txt"
echo "$MEM" > "$RAW/mem_$TIME.txt"
echo "$DISK" > "$RAW/disk_$TIME.txt"

#Saving in log file
echo "$(date): collected CPU=$CPU% MEM=$MEM% DISK=$DISK%" >> "$LOG"
```

### Explanation
vmstat → gets CPU idle time → calculate usage as 100 - $15

free -m → shows used/total memory → converted to percent

df -P / → prints disk usage of root /

Files saved as:

cpu_YYYY-MM-DD-HH-MM.txt

mem_YYYY-MM-DD-HH-MM.txt

disk_YYYY-MM-DD-HH-MM.txt

Cron Job:
```bash
sudo crontab -e
0 * * * * /opt/sysstats/collect.sh
```
_____________________________________________

## 🧮 Section 2 – Calculating Averages

**Question:**
Create another script that reads all collected files, calculates averages, and saves them in /opt/sysstats/avg/.

**Answer:**

```bash
#!/bin/bash

# Folders
RAW=/opt/sysstats/raw
AVG=/opt/sysstats/avg/
LOG=/opt/sysstats/logs/avg.log

# calculating the avg
function avg_of(){
    awk '{sum+=$1; count++} END {if (count>0) printf("%.2f", sum/count); else print "0"}' $RAW/$1_*.txt
}

# using the function to get the avg of each task
CPU_AVG=$(avg_of cpu)
MEM_AVG=$(avg_of mem)
DISK_AVG=$(avg_of disk)

# Displaying the AVG's
echo "$CPU_AVG" > "$AVG/cpu_avg.txt"
echo "$MEM_AVG" > "$AVG/mem_avg.txt"
echo "$DISK_AVG" > "$AVG/disk_avg.txt"

# Displaying the AVG's in the log
echo "$(date): Avg CPU=$CPU_AVG% MEM=$MEM_AVG% DISK=$DISK_AVG%" >> "$LOG"
```

### Within the same file (avg.sh) this section is for **Displaying Results on a Web Page via Apache Server**.

### but before doing anything we should start by:

### Installing and enabling Apache
```bash
sudo dnf install -y httpd 
sudo systemctl enable --now httpd # start and enable Apache
```
### Opening HTTP Port 80 (Firewall)
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

#### Verify Apache is running
```bash
sudo systemctl status httpd
curl -I http://127.0.0.1
```
_____________________________________________________________

```bash
WEB=/var/www/html/sysstats
mkdir -p "$WEB"


# copy the average numbers from /opt/sysstats/avg/ to WEB folder
cp -f "$AVG/cpu_avg.txt" "$WEB/"
cp -f "$AVG/mem_avg.txt" "$WEB/"
cp -f "$AVG/disk_avg.txt" "$WEB/"

#CPU Displaying part for HTML

cat > "$WEB/cpu.html" <<HTML
<html>
  <body>
    <h1>CPU Usage</h1>
    <p><b>Average:</b> $(cat "$AVG/cpu_avg.txt")%</p>

    <h2>Last 10 records</h2>
    <table border="1">
      <tr><th>Timestamp</th><th>CPU %</th></tr>
$(for file in $(ls -t $RAW/cpu_*.txt | head -10)
do
   time=$(basename "$file" .txt | cut -d'_' -f2)
   value=$(cat "$file")
   echo "<tr><td>$time</td><td>$value%</td></tr>"
done)
    </table>

    <p><a href="index.html">Back</a></p>
  </body>
</html>
HTML

#Memory Displaying part for HTML

cat > "$WEB/mem.html" <<HTML
<html>
  <body>
    <h1>Memory Usage</h1>
    <p><b>Average:</b> $(cat "$AVG/mem_avg.txt")%</p>

    <h2>Last 10 records</h2>
    <table border="1">
      <tr><th>Timestamp</th><th>Memory %</th></tr>
$(for file in $(ls -t $RAW/mem_*.txt | head -10)
do
   time=$(basename "$file" .txt | cut -d'_' -f2)
   value=$(cat "$file")
   echo "<tr><td>$time</td><td>$value%</td></tr>"
done)
    </table>

    <p><a href="index.html">Back</a></p>
  </body>
</html>
HTML

#Disk Displaying part for HTML

cat > "$WEB/disk.html" <<HTML
<html>
  <body>
    <h1>Disk Usage</h1>
    <p><b>Average:</b> $(cat "$AVG/disk_avg.txt")%</p>

    <h2>Last 10 records</h2>
    <table border="1">
      <tr><th>Timestamp</th><th>Disk %</th></tr>
$(for file in $(ls -t $RAW/disk_*.txt | head -10)
do
   time=$(basename "$file" .txt | cut -d'_' -f2)
   value=$(cat "$file")
   echo " <tr><td>$time</td><td>$value%</td></tr>"
done)
    </table>

    <p><a href="index.html">Back</a></p>
  </body>
</html>
HTML
```



