# Web Server Log Analysis using Apache Hive

## Project Overview
This project analyzes web server logs using Apache Hive. The dataset consists of log entries with details such as IP address, timestamp, requested URL, HTTP status code, and user agent. The goal is to extract meaningful insights and optimize query performance using partitioning.

## Implementation Approach
The analysis is performed using HiveQL queries for various tasks:

### 1. Count Total Web Requests
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/web_logs_analysis/total_requests'
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
SELECT 'Total Requests:', COUNT(*) FROM web_logs;
```

### 2. Analyze Status Codes
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/web_logs_analysis/status_analysis'
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
SELECT analysis, status, cnt 
FROM (
    SELECT 'Status Code Analysis:' AS analysis, status, COUNT(*) AS cnt
    FROM web_logs
    GROUP BY status
) t
ORDER BY cnt DESC;
```

### 3. Identify Most Visited Pages
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/web_logs_analysis/most_visited'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT 'Most Visited Pages:', url, cnt
FROM (
    SELECT url, COUNT(*) AS cnt
    FROM web_logs
    GROUP BY url
    ORDER BY cnt DESC
    LIMIT 3
) t;
```

### 4. Traffic Source Analysis
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/web_logs_analysis/traffic_source'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT 'Traffic Source Analysis:', user_agent, cnt
FROM (
    SELECT user_agent, COUNT(*) AS cnt
    FROM web_logs
    GROUP BY user_agent
    ORDER BY cnt DESC
) t;
```

### 5. Detect Suspicious IPs
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/web_logs_analysis/suspicious_ips'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT 'Suspicious IPs:', ip, failed_requests
FROM (
    SELECT ip, COUNT(*) AS failed_requests
    FROM web_logs
    WHERE status IN (404, 500)
    GROUP BY ip
    HAVING COUNT(*) > 3
) t;
```

### 6. Analyze Traffic Trends Over Time
```sql
SELECT SUBSTR(timestamp, 1, 16) AS minute, COUNT(*) AS request_count
FROM web_logs
GROUP BY SUBSTR(timestamp, 1, 16)
ORDER BY minute;
```

### 7. Implement Partitioning
To optimize query performance, we partitioned the table by HTTP status code.

#### Creating a Partitioned Table
```sql
CREATE TABLE IF NOT EXISTS web_logs_partitioned (
    ip STRING,
    timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

#### Loading Data into Partitioned Table
```sql
SET hive.exec.dynamic.partition.enabled=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE web_logs_partitioned PARTITION (status)
SELECT ip, timestamp, url, user_agent, status FROM web_logs;
```
## Execution Steps
Here are the steps for processing and analyzing the web server log file using Apache Hive in Hue:

### 1️⃣ Copy the CSV File into the Namenode Container
Since HDFS runs inside Docker containers, first copy the file into the Namenode container:
```sh
docker cp web_server_logs.csv namenode:/web_server_logs.csv
```

### 2️⃣ Upload the File to HDFS
Log into the Namenode container:
```sh
docker exec -it namenode /bin/bash
```
Then, move the file into HDFS:
```sh
hdfs dfs -put /web_server_logs.csv /user/hive/warehouse/
```
Verify that the file is successfully uploaded:
```sh
hdfs dfs -ls /user/hive/warehouse/
```

### 3️⃣ Create Hive Database and Table in Hue
Log into Hue and create a Hive database:
```sql
CREATE DATABASE IF NOT EXISTS web_logs_db;
```
Use the created database:
```sql
USE web_logs_db;
```
Create an external table for web server logs:
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs (
    ip STRING,
    timestamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/';
```

### 4️⃣ Load Data into Hive Table
Run the following command in Hue:
```sql
LOAD DATA INPATH '/user/hive/warehouse/web_server_logs.csv' INTO TABLE web_logs;
```


## Challenges Faced
- **Error: Dynamic Partitioning Strict Mode**: Resolved by setting `hive.exec.dynamic.partition.mode=nonstrict`.
- **String Manipulation in Queries**: Adjusted timestamp processing using `SUBSTR()`.
- **Output Format Issues**: Ensured proper text formatting while writing results.

## Sample Input and Output
### Sample Log Data (CSV Format)
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

### Expected Output
```
Total Requests: 100

Status Code Analysis:
200: 80
404: 10
500: 10

Most Visited Pages:
/home: 50
/products: 30
/checkout: 20

Traffic Source Analysis:
Mozilla/5.0: 60
Chrome/90.0: 30
Safari/13.1: 10

Suspicious IP Addresses:
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests

Traffic Trend Over Time:
2024-02-01 10:15: 5 requests
2024-02-01 10:16: 7 requests
```
### **Partition Information**
To list all partitions:
```sql
SHOW PARTITIONS web_logs_partitioned;
```
Output:
```
status=200
status=404
status=500
```

To view records from the partition where `status = 404`:
```sql
SELECT * FROM web_logs_partitioned WHERE status = 404;
```


This README provides a structured guide for setting up and executing the analysis in Apache Hive.

