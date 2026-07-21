### 1. CPU QUERIES

#### 1.1 Total CPU Cores

##### Query:

```
count(node_cpu_seconds_total{mode="idle"})
```

##### Meaning:

Total number of CPU cores in the system

##### Explanation:

- `node_cpu_seconds_total` → CPU metric per core
- `{mode="idle"}` → select one metric per CPU
- `count()` → counts number of CPUs

##### Expected Output:

```
4
```

System has 4 CPU cores

---

#### 1.2 CPU Usage %

##### Query:

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

##### Meaning:

Percentage of CPU being used

##### Explanation:

- `mode="idle"` → CPU idle time
- `rate(...[1m])` → idle change per second
- `avg()` → average across CPUs
- `100 -` → convert idle → usage

##### Expected Output:

```
65.2
```

CPU is 65% utilized

---

#### 1.3 CPU Idle %

##### Query:

```
avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100
```

##### Meaning:

Percentage of CPU idle time

##### Expected Output:

```
34.8
```

---

### 2. MEMORY QUERIES

#### 2.1 Total Memory

##### Query:

```
node_memory_MemTotal_bytes
```

##### Meaning:

Total RAM in bytes

##### Explanation:

- Direct metric from Node Exporter

##### Expected Output:

```
16777216000
```

~16 GB

---

#### 2.2 Available Memory

##### Query:

```
node_memory_MemAvailable_bytes
```

##### Meaning:

Free memory available for use

##### Expected Output:

```
4000000000
```

~4 GB free

---

#### 2.3 Used Memory %

##### Query:

```
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

##### Meaning:

Percentage of used memory

##### Explanation:

- `MemAvailable / MemTotal` → free ratio
- `1 -` → used ratio

##### Expected Output:

```
75.0
```

75% memory used

---

### 3. DISK QUERIES

#### 3.1 Total Disk Space

##### Query:

```
sum(node_filesystem_size_bytes{fstype!="tmpfs"})
```

##### Meaning:

Total disk size

##### Explanation:

- `sum()` → total across disks
- `fstype!="tmpfs"` → ignore temporary filesystems

##### Expected Output:

```
500000000000
```

~500 GB

---

#### 3.2 Available Disk Space

##### Query:

```
sum(node_filesystem_avail_bytes{fstype!="tmpfs"})
```

##### Meaning:

Available disk space

---

#### 3.3 Used Disk %

##### Query:

```
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

##### Meaning:

Disk usage percentage

##### Expected Output:

```
80.5
```

---

### 4. NETWORK QUERIES

#### 4.1 Incoming Traffic

##### Query:

```
rate(node_network_receive_bytes_total[1m])
```

##### Meaning:

Incoming network traffic per second

##### Explanation:

- `receive_bytes_total` → total received data
- `rate()` → converts to per second

##### Expected Output:

```
125000
```

~125 KB/sec

---

#### 4.2 Outgoing Traffic

##### Query:

```
rate(node_network_transmit_bytes_total[1m])
```

##### Meaning:

Outgoing traffic per second

---

### 5. IMPORTANT FUNCTIONS

#### 5.1 rate()

##### Query Example:

```
rate(node_network_receive_bytes_total[1m])
```

##### Meaning:

Calculates per-second change

##### Use:

- Network speed
- Disk I/O
- CPU changes

---

#### 5.2 sum()

##### Query Example:

```
sum(node_filesystem_size_bytes)
```

##### Meaning:

Adds all values

##### Use:

- Total disk
- Total traffic

---

#### 5.3 avg()

##### Query Example:

```
avg(rate(node_cpu_seconds_total[1m]))
```

##### Meaning:

Average value

##### Use:

- CPU average across cores

---

#### 5.4 count()

##### Query Example:

```
count(node_cpu_seconds_total{mode="idle"})
```

##### Meaning:

Counts number of elements

##### Use:

- CPU cores
- Instances

---

### 6. TIME-BASED QUERIES

#### 6.1 Last 5 Minutes Data

##### Query:

```
rate(node_cpu_seconds_total[5m])
```

##### Meaning:

Uses last 5 minutes data

---

#### 6.2 Specific Time Data

##### Query:

```
node_memory_MemAvailable_bytes @ 1712400000
```

##### Meaning:

Fetch data at exact timestamp

##### Explanation:

- `@` → specific time

---

#### 6.3 Historical Range

##### Query:

```
node_cpu_seconds_total[1h]
```

##### Meaning:

Returns data for last 1 hour

---

### 7. BONUS (VERY USEFUL)

#### System Load

```
node_load1
```

System load

#### Disk Read Speed

```
rate(node_disk_read_bytes_total[1m])
```

#### Disk Write Speed

```
rate(node_disk_written_bytes_total[1m])
```
