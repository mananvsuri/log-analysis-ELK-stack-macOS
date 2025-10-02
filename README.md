## Step 1: Install Tools 🛠️

First, open your Terminal. We'll install Homebrew and then use it to install Docker.

Bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Docker Desktop
brew install --cask docker
After this finishes, open Docker Desktop from your Applications folder and make sure it's running. Check its Settings > Resources and give it at least 4 GB of Memory.

## Step 2: Create Your Project Folder 📁

Now, let's create a clean folder for all our files.

Bash
# Create all the needed directories and navigate inside
mkdir -p ~/Desktop/Cyber_Analysis_Project/logstash/pipeline logs
cd ~/Desktop/Cyber_Analysis_Project
## Step 3: Create the Docker Compose File

This file is the main blueprint that tells Docker which programs to install and run. Notice how it lists elasticsearch, logstash, and kibana.

Bash
cat << EOF > docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.9.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr_share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.9.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logs:/usr/share/logstash/logs:ro
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.9.0
    container_name: kibana
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
    driver: local
EOF
## Step 4: Create the Logstash Config

This file tells Logstash how to understand our logs.

Bash
cat << EOF > logstash/pipeline/logstash.conf
input {
  file {
    path => "/usr/share/logstash/logs/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => { "message" => "%{COMMONAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logstash-access-%{+YYYY.MM.dd}"
  }
}
EOF
## Step 5: Create the Log File

This is the actual evidence file for our investigation.

Bash
cat << EOF > logs/access.log
203.0.113.54 - - [01/Oct/2025:23:10:15 +0530] "GET /portal/login.php HTTP/1.1" 200 1543
78.45.12.101 - - [01/Oct/2025:23:11:05 +0530] "GET /portal/css/main.css HTTP/1.1" 200 8764
188.114.97.8 - - [01/Oct/2025:23:12:30 +0530] "GET /portal/admin.php HTTP/1.1" 404 152
188.114.97.8 - - [01/Oct/2025:23:12:35 +0530] "GET /portal/studentProfile.php?id=123 HTTP/1.1" 200 3451
188.114.97.8 - - [01/Oct/2025:23:13:10 +0530] "GET /portal/studentProfile.php?id=123' HTTP/1.1" 500 505
203.0.113.54 - - [01/Oct/2025:23:14:22 +0530] "POST /portal/login.php HTTP/1.1" 200 4311
188.114.97.8 - - [01/Oct/2025:23:14:50 +0530] "GET /portal/studentProfile.php?id=123 AND 1=2 HTTP/1.1" 200 1254
188.114.97.8 - - [01/Oct/2025:23:15:05 +0530] "GET /portal/studentProfile.php?id=123' OR '1'='1 HTTP/1.1" 500 505
188.114.97.8 - - [01/Oct/2025:23:15:42 +0530] "GET /portal/studentProfile.php?id=123' UNION SELECT 1,username,password FROM users-- HTTP/1.1" 200 4587
188.114.97.8 - - [01/Oct/2025:23:15:55 +0530] "GET /portal/studentProfile.php?id=123' UNION SELECT 1,cc_number,cc_expiry FROM credit_cards-- HTTP/1.1" 200 4612
92.115.34.12 - - [01/Oct/2025:23:16:15 +0530] "GET /portal/index.php HTTP/1.1" 200 2876
EOF
## Step 6: Launch the ELK Stack 🚀

Now, run the command that brings everything to life.

Bash
docker compose up
This is the step that installs and runs the entire ELK Stack. Docker reads your docker-compose.yml file, sees the list of services (elasticsearch, logstash, kibana), downloads their official software packages (called images), and runs all of them together. It's an all-in-one command for installation and execution.

Wait a few minutes for it to download and start. Leave this terminal window running.

## Step 7: Perform the Analysis in Kibana 🕵️

Open your browser to http://localhost:5601.

Go to the Menu (☰) > Analytics > Discover.

Create a Data View named logstash-* (use @timestamp as the time field).

Set the Time Filter in the top right to Absolute and select 2025-10-01 23:10:00 to 2025-10-01 23:20:00.

In the search bar, type "UNION SELECT" and press Enter.

Expand a result, find the source.address field (188.114.97.8), and click the + icon to filter for it.

Clear "UNION SELECT" from the search bar to see the attacker's full history.
