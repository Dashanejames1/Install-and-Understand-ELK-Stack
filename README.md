# Install-and-Understand-ELK-Stack


**Author:** Dashane James  
**Lab Environment:** VMware Workstation | Kali Linux | ELK Stack  
**Purpose:** Install and configure the ELK Stack on Kali Linux, verify each component is running, and document the role of each in a security monitoring and log analysis context.  
**Status:** 🔵 Completed

---

## 📋 Overview

The ELK Stack, Elasticsearch, Logstash, and Kibana, is one of the most widely deployed open-source SIEM and log analysis platforms in enterprise security environments. Security teams use it to ingest logs from endpoints, network devices, and applications; index them for fast searching; and surface events through dashboards and alerts. Understanding how the three components interact, and where Beats fit in, is foundational to any detection engineering or SOC analyst role.

This lab covers installing and starting all three components on Kali Linux, verifying Elasticsearch is responding on its expected port, accessing Kibana's web interface and exploring the Discover section, and documenting the function of each component in the overall data pipeline.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Analyst Machine | Kali Linux 2026.1, `192.168.79.129` |
| Target Machine | N/A (this lab focuses on tooling setup) |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11, ASUS Vivobook 14 |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment. The ELK Stack was installed locally on the Kali machine for log analysis and detection engineering practice.

---

## 🛠️ Tools Used

- **Elasticsearch:** Distributed search and analytics engine; stores log data as indexed JSON documents and exposes a REST API on port 9200. Every other component in the stack communicates with it through that API.
- **Logstash:** Data processing pipeline; ingests logs from configured input sources, applies parsing and transformation rules in the filter stage, and outputs structured data to Elasticsearch. Pipeline behavior is defined in `.conf` files with input, filter, and output sections.
- **Kibana:** Web-based visualization and search interface for Elasticsearch; accessible at `http://localhost:5601`. The Discover section provides raw event search, dashboards allow aggregated views across time ranges, and the alerting engine fires notifications when detection queries match.
- **Beats (reference only, not installed in this lab):** Lightweight log shipper agents that run on monitored hosts and forward data to Logstash or directly to Elasticsearch. Filebeat handles log files; Winlogbeat handles Windows Event Logs; Auditbeat handles Linux audit framework events. They are the standard collection layer in production deployments.

---

## 🔬 Tasks / Assessments Performed

### 1. Install ELK on Kali Linux

All three ELK components are available through Kali's package manager as individual packages.

```bash
# Update package lists first
sudo apt update

# Install Elasticsearch, Logstash, and Kibana
sudo apt install elasticsearch logstash kibana -y

# Start Elasticsearch and Kibana
sudo systemctl start elasticsearch
sudo systemctl start kibana

# Optional: enable both services to start automatically on reboot
sudo systemctl enable elasticsearch
sudo systemctl enable kibana
```

Logstash is installed as a dependency and library but not started here because no input pipeline has been configured yet. Starting it without a pipeline defined causes it to run in a wait state without doing any processing.

**Service startup timing:** Elasticsearch takes 30 to 60 seconds to fully initialize after `systemctl start`. Running `curl http://localhost:9200` immediately after the start command usually returns a connection refused error. Wait 30 to 60 seconds before attempting the verification step.

**Common installation issue: Elasticsearch 8.x security defaults**

Elasticsearch 8.x ships with security features enabled. By default it expects HTTPS connections and requires authentication, which causes `curl http://localhost:9200` to return an SSL error or prompt for credentials. In an isolated local lab environment, disabling these features is the practical path forward.

```bash
# Edit the main Elasticsearch configuration file
sudo nano /etc/elasticsearch/elasticsearch.yml

# Add or set these two lines:
xpack.security.enabled: false
xpack.security.http.ssl.enabled: false

# Restart Elasticsearch after saving the change
sudo systemctl restart elasticsearch
```

After this change, `curl http://localhost:9200` works over plain HTTP without credentials.

**Common installation issue: insufficient memory**

Elasticsearch reserves a significant portion of available RAM at startup. On a VM with limited memory, it may start slowly or fail entirely. If `systemctl status elasticsearch` reports a failed state, check the service logs:

```bash
sudo journalctl -u elasticsearch -n 50
```

An out-of-memory error in the log means the JVM heap allocation exceeds available RAM. Reduce it by editing `/etc/elasticsearch/jvm.options` and setting both heap values to 512m:

```
-Xms512m
-Xmx512m
```

Restart Elasticsearch after making this change. This is sufficient for a single-node lab instance with no production workload.

---

### 2. Verify Elasticsearch is running on port 9200

```bash
curl http://localhost:9200
```

A successful response confirms Elasticsearch is running, reachable on port 9200, and accepting API requests:

```json
{
  "name" : "kali",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xK9mZrQfTpS3aBnNvUlEwA",
  "version" : {
    "number" : "8.x.x",
    "build_flavor" : "default",
    "build_type" : "deb",
    "lucene_version" : "9.x.x",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

The `"You Know, for Search"` tagline has been in every Elasticsearch response since version 1.0. It confirms the node is healthy and ready to accept index and query operations. The cluster name defaults to `elasticsearch` and the node name defaults to the hostname of the machine.

Additional verification commands:

```bash
# Check cluster health (green = all shards allocated and healthy)
curl http://localhost:9200/_cluster/health?pretty

# List all indexes currently stored in Elasticsearch
curl http://localhost:9200/_cat/indices?v

# Check system service status
sudo systemctl status elasticsearch
```

**Finding:** Port 9200 is the HTTP REST API used by every component and tool that communicates with Elasticsearch. Port 9300 handles internal cluster communication between nodes, which is not relevant in a single-node lab setup but is an important port to restrict in production. Exposing port 9200 to the internet without authentication is one of the most common and consequential misconfigurations in cloud-deployed ELK stacks. Dozens of large-scale data breaches have involved unprotected Elasticsearch nodes reachable from the public internet.

---

### 3. Access Kibana and explore the Discover section

Open a browser on the Kali machine and navigate to `http://localhost:5601`. With Elasticsearch security disabled as documented in Task 1, Kibana connects to the local Elasticsearch instance automatically and opens without a login prompt.

**Navigating to Discover:**

From the Kibana home page: hamburger menu (top-left) → Analytics → Discover.

Discover is Kibana's primary interface for searching and exploring raw indexed events. It requires an index pattern before any data is visible. An index pattern tells Kibana which Elasticsearch indexes to query and which field represents the event timestamp.

**Creating an index pattern:**

With no data ingested yet, Discover shows an empty state. In a populated environment, index patterns are created by navigating to Stack Management → Index Patterns → Create index pattern, entering a pattern that matches the index naming convention in use (such as `logstash-*` for Logstash output, or `filebeat-*` for Filebeat), and selecting the timestamp field.

**What the Discover section provides once logs are flowing:**

After an index pattern is configured and data is being ingested, the Discover interface provides:

- A histogram showing event volume over the selected time range, making activity spikes immediately visible
- A document table with one row per indexed event, expandable to show all fields and their values
- A field sidebar listing every indexed field and its data type, with click-to-filter functionality
- A KQL (Kibana Query Language) search bar for filtering events by field value, pattern, or time range

Example KQL queries for security log analysis once data is flowing:

```
# Filter events from a specific source IP
source.ip: "192.168.79.129"

# Find all process creation events
event.category: "process"

# Find events tagged with a specific ATT&CK technique
tags: "attack.t1190"

# Find FTP login attempts containing the vsftpd backdoor trigger pattern
ftp.username: *:)*
```

**Finding:** The Discover interface is only as useful as the data flowing into it. The depth and specificity of what can be searched depends entirely on how well logs were parsed and structured at the Logstash or Beats layer. A field like `ftp.username` only exists as a searchable field in Discover if Logstash was configured to parse it out of the raw FTP log text. This is the direct connection between pipeline design decisions and detection capability.

---

### 4. Document the role of each ELK component

**Elasticsearch: Store and Index**

Elasticsearch is a distributed document store built on Apache Lucene. Each log event arrives as a JSON document and is stored inside a named index. When Logstash or Beats ships a log event, Elasticsearch maps each field to the appropriate data type, builds inverted indexes across all fields, and makes the document searchable in near real time regardless of how large the index has grown.

Key concepts: an index is a collection of documents; shards are how an index is split across nodes in a cluster; mappings define the data types for each field; the REST API is how every other component communicates with Elasticsearch. In a single-node lab, sharding is not relevant, but the API and mappings matter the moment data starts flowing.

**Logstash: Ingest and Parse**

Logstash is a configurable data pipeline with three stages: input, filter, and output. The input section defines where logs come from, such as a Beats forwarder on TCP 5044, a syslog listener, or a file path. The filter section is where raw log text is transformed into structured fields using parsing plugins like `grok` for plain-text log formats or `json` for already-structured input. The output section sends the structured events to their destination, almost always Elasticsearch in a security context.

The filter stage is where the analytical value is created. A raw Apache access log line like `192.168.79.129 - - [25/Oct/2026:09:15:42 +0000] "GET /admin HTTP/1.1" 200 1234` becomes individual, independently searchable fields (client IP, timestamp, HTTP method, URL path, response code, bytes transferred) that Kibana and detection rules can query separately. Without the filter stage, the entire line lands in Elasticsearch as a single unstructured string and most detection logic cannot be applied to it.

**Kibana: Visualize and Search**

Kibana is the web interface layer. It connects to Elasticsearch and translates complex query logic into a usable search bar, dashboard tiles, and alert rules. For security work, the three most relevant capabilities are: Discover (search and explore raw events, filter by field, export data), Dashboards (aggregate views showing event counts, top sources, timeline charts, and geographic data), and Alerts (rule-based notifications that fire when a detection query returns results).

Kibana Security, available in higher license tiers, adds pre-built SIEM views aligned to ATT&CK phases. The free open-source version still supports custom dashboards and alerting with manually written KQL detection rules.

**Beats: Ship**

Beats are not installed in this lab but represent the standard data collection layer in production. Each Beat is a lightweight agent that runs on a host, watches a specific log source, and forwards events to Logstash or directly to Elasticsearch. Filebeat reads log files line by line and ships new entries as they are written. Winlogbeat watches the Windows Event Log channels and ships security-relevant events such as logon events (Event ID 4624), logon failures (Event ID 4625), and process creation (Event ID 4688). Auditbeat reads Linux auditd events and ships process execution, file access, and socket activity. The lightweight design means Beats can run on nearly any host with minimal CPU and memory impact.

**Full data pipeline:**

```
Log sources → Beats (lightweight shippers)
                     ↓
              Logstash (parse and transform)
                     ↓
           Elasticsearch (store and index)
                     ↓
              Kibana (search and visualize)
```

In simplified deployments, Logstash can be bypassed when Beats' built-in processors or Elasticsearch ingest pipelines handle the parsing work:

```
Log sources → Beats → Elasticsearch → Kibana
```

---

## 📊 Key Findings Summary

| Component | Port | Role | Status |
|---|---|---|---|
| Elasticsearch | 9200 (HTTP REST API), 9300 (cluster transport) | Store and index all log documents | 🟢 Running, verified via `curl http://localhost:9200` |
| Kibana | 5601 | Web interface for search, dashboards, and alerts | 🟢 Running, Discover section accessible |
| Logstash | 5044 (Beats input, when configured) | Ingest pipeline: parse raw log text into structured, searchable fields | 🔵 Installed, not started (no pipeline configured) |
| Filebeat | ships from monitored hosts | Forward log files to Logstash or Elasticsearch | 🔵 Reference only (not installed in this lab) |
| Winlogbeat | ships from Windows hosts | Forward Windows Event Log data to Logstash or Elasticsearch | 🔵 Reference only (not installed in this lab) |

---

## 🗺️ MITRE ATT&CK Mapping

The ELK Stack is a defensive detection platform. The relevant ATT&CK framing for this lab is what adversary techniques each log source the stack ingests would enable detection of, connecting the offensive behavior documented in Week 2 to the detection infrastructure built here.

| ELK Log Source | ATT&CK Data Component | Adversary Techniques Detected |
|---|---|---|
| FTP service logs via Filebeat | Network Traffic: Network Connection Creation | T1190 (vsftpd backdoor trigger via `:)` in FTP USER field) |
| Process creation logs via Auditbeat | Process: Process Creation | T1059.004 (shell execution via Meterpreter), T1033, T1082 (post-exploitation discovery commands) |
| File access logs via Auditbeat | File: File Access | T1087.001 (`/etc/passwd` read), T1003.008 (`/etc/shadow` read) |
| Authentication logs via Filebeat | User Account: User Authentication | T1110 (brute force: multiple failed logins), T1078 (valid accounts: successful logins from unexpected sources) |

**Connection to Week 2:** The five Sigma detection rules written in the Post-Exploitation-Enumeration lab can be converted directly to Kibana KQL alert queries. The detection logic is identical; only the query syntax changes to match Elasticsearch field naming conventions. The ATT&CK technique IDs used as Sigma rule tags become the search terms used to audit detection coverage in Kibana.

---

## 🛡️ Defensive Recommendations

1. **Keep Elasticsearch security enabled in any non-lab environment (Critical):** Elasticsearch 8.x enables TLS and authentication by default for good reason. The practice of disabling it for convenience, then forgetting to re-enable it before deployment, has been the direct cause of large-scale data breaches involving millions of exposed records. In lab, disable it. In staging or production, leave it enabled and configure proper credentials and certificate management.

2. **Restrict port 9200 to localhost or a trusted management network (High):** Even with authentication enabled, Elasticsearch's REST API should not be reachable from outside the host or trusted monitoring segment. A firewall rule limiting inbound connections to port 9200 from localhost only eliminates the largest attack surface for unauthorized data access.

3. **Use Beats agents on monitored hosts instead of remote log forwarding where possible (Medium):** Beats read and parse log data at the source host before shipping it, which reduces Logstash processing load, ensures consistent field naming, and captures log data even when the Logstash or Elasticsearch node is temporarily unavailable (Beats will buffer and retry).

4. **Define index lifecycle management policies before ingesting data (Medium):** Without a retention policy, Elasticsearch indexes grow indefinitely. A disk-full node stops accepting writes and effectively takes the SIEM offline. Define ILM policies at initial setup to automatically roll over large indexes and delete data older than the required retention window.

5. **Tag every detection rule with the ATT&CK technique ID it targets (Medium):** Consistent tagging across Kibana alert rules, Sigma rules, and SIEM dashboards makes it possible to build a coverage matrix showing which techniques have detection logic and which do not. Gaps in the matrix are gaps in visibility. This practice applies regardless of whether the platform is ELK, Splunk, or any other SIEM.

---

## 🔑 Technical Notes

```bash
# Install ELK Stack on Kali
sudo apt update && sudo apt install elasticsearch logstash kibana -y

# Start services
sudo systemctl start elasticsearch
sudo systemctl start kibana

# Check service status
sudo systemctl status elasticsearch
sudo systemctl status kibana

# Verify Elasticsearch is running (wait 30-60 seconds after start)
curl http://localhost:9200

# Check cluster health
curl http://localhost:9200/_cluster/health?pretty

# List all indexes
curl http://localhost:9200/_cat/indices?v

# Access Kibana
# Open browser on Kali: http://localhost:5601

# Disable Elasticsearch security features (lab/isolated environments only)
# Edit /etc/elasticsearch/elasticsearch.yml:
#   xpack.security.enabled: false
#   xpack.security.http.ssl.enabled: false
# Then restart: sudo systemctl restart elasticsearch

# Fix memory-related startup failure
# Edit /etc/elasticsearch/jvm.options:
#   -Xms512m
#   -Xmx512m
# Then restart: sudo systemctl restart elasticsearch

# View service logs for troubleshooting
sudo journalctl -u elasticsearch -n 50
sudo journalctl -u kibana -n 50
```

**ELK port reference:**

| Port | Service | Purpose |
|---|---|---|
| 9200 | Elasticsearch | HTTP REST API (primary interface for all components and tools) |
| 9300 | Elasticsearch | Transport layer for inter-node cluster communication |
| 5601 | Kibana | Web UI |
| 5044 | Logstash | Beats input plugin (default port for Filebeat/Winlogbeat forwarding) |
| 514 | Logstash | Syslog input, UDP/TCP (when configured) |

**ELK vs. Splunk:**

Both platforms address the same core problem: centralized log collection, indexed search, and event-driven alerting. The practical differences are scope and configuration overhead. ELK is open source and requires manual setup of every pipeline stage, index pattern, and alert rule. Splunk's free tier handles most of this configuration automatically but caps daily data ingestion at 500MB. Detection concepts (field extraction, query logic, threshold alerts, ATT&CK mapping) transfer directly between the two platforms. Working with both is more valuable than working with either one alone.

---

## 📌 About This Project

This repository documents the second SIEM platform built during the SIEM, log analysis, and incident response phase. The first three repositories set up Splunk, practiced SPL, and generated real attack traffic to detect. This one installs ELK as an alternative detection platform, documents how its pipeline works from log collection through Kibana visualization, and establishes the technical grounding needed to work with either platform in a production SOC environment.

The most important concept here is the pipeline. A log event flowing from a monitored host through Beats to Logstash to Elasticsearch to Kibana passes through four distinct processing stages, and understanding what happens at each stage is what makes the difference between an analyst who can run a search and one who can diagnose why a detection rule is not firing. The Sigma detection rules written in the previous phase's post-exploitation lab become Kibana KQL alert queries here. The ATT&CK technique IDs referenced throughout the portfolio become the audit terms used to assess detection coverage across the stack.

**Related repositories (SIEM, log analysis, and incident response phase):**
- `Splunk-Free-Installation-and-Ingest-First-Logs`: Installed Splunk Enterprise and ingested the first log files, establishing the primary SIEM used throughout the detection labs
- `Splunk-Search-Language`: Core SPL commands practiced and applied as the detection query language for the Splunk SIEM
- `Generating-Attacks-and-Defending-in-Splunk`: Generated live attack traffic (port scan and SSH brute force), ingested it into Splunk, and built automated detection alerts for both attack types
- `Install-Snort-IDS-and-Write-Detection-Rules`: Installed Snort 3, wrote a custom ICMP detection rule, and confirmed it fired on live traffic from Metasploitable
- `Incident-Response-Simulation`: Walked through all seven NIST SP 800-61 phases using the SSH brute force attack as the scenario, from preparation through lessons learned
- `Incident-Response-Report`: Formal incident report for the SSH brute force event, covering timeline, impact assessment, containment, root cause, and recommended controls

---

## 👤 Author

**Dashane James**
Senior Field Service Technician → Cybersecurity Analyst
📍 Yonkers, NY
🎓 B.S. Information Technology, SUNY Canton
🏆 CompTIA Security+
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*
