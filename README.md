Local Log Analysis with ELK Stack on macOS
This guide provides a complete walkthrough for setting up a local ELK (Elasticsearch, Logstash, Kibana) stack using Docker on macOS to analyze web server access logs for a simulated SQL injection attack.

Prerequisites
macOS: With an Intel or Apple Silicon processor.

Homebrew: The missing package manager for macOS. If you don't have it, install it first:

Bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
Docker Desktop: A tool for running applications in isolated containers.

Setup Instructions
1. Install and Configure Docker

First, install Docker Desktop via Homebrew, then launch it and configure its resources.

Bash
# Install Docker Desktop
brew install --cask docker

# After installation, open Docker from your Applications folder.
# Click the Docker icon in the menu bar > Settings... > Resources.
# Ensure Memory is set to a minimum of 4 GB.
2. Create Project Structure

Create a dedicated directory to hold all configuration files and log data.

Bash
# Create the full directory structure
mkdir -p ~/Desktop/Cyber_Analysis_Project/logstash/pipeline logs

# Navigate into the project's root directory
cd ~/Desktop/Cyber_Analysis_Project
3. Define Services with Docker Compose

Create a docker-compose.yml file in the project root. This file defines the three services of the ELK stack.

YAML
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
      - es_data:/usr/share/elasticsearch/data

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
4. Configure the Logstash Pipeline

Create a logstash.conf file inside the logstash/pipeline directory. This configuration tells Logstash how to process the log file.

Code snippet
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
5. Add the Log Data

Create the access.log file inside the logs directory. This file contains the sample data for our investigation.

Code snippet
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
Running the Stack
With all files in place, launch the ELK stack from the root of your project directory (~/Desktop/Cyber_Analysis_Project).

Bash
docker compose up
This command will download the required images and start all three containers. This may take several minutes on the first run. Leave this terminal window running.

Performing the Analysis in Kibana
Access Kibana: Open a web browser and navigate to http://localhost:5601.

Create Data View:

Click the main menu (☰) > Analytics > Discover.

You will be prompted to create a data view.

Name: logstash-*

Timestamp field: @timestamp

Click Create data view.

Isolate the Incident:

In the top-right corner, click the time filter. Select the Absolute tab.

Set the range from 2025-10-01 23:10:00 to 2025-10-01 23:20:00 and click Update.

In the search bar, type "UNION SELECT" and press Enter. This will show you the successful attack logs.

Investigate the Attacker:

Expand one of the log entries.

Find the source.address field (188.114.97.8) and click the + icon to filter for this value.

Clear the "UNION SELECT" text from the search bar and press Enter.

This final view shows the complete, chronological list of all actions performed by the attacker.

Shutdown and Cleanup
When you are finished, you can shut down the stack and remove the containers and data volume.

Bash
# In the terminal where the stack is running, press Control + C.
# Then run the following commands to clean up.
docker compose down
docker volume rm Cyber_Analysis_Project_es_data
