# Task 5: Incident Response Procedures
## Detection, Containment, & Recovery Plan

---

## Incident Response Framework

### NIST Incident Response Phases

```
Phase 1: Preparation
├─ Tools deployed
├─ Processes documented
├─ Team trained
└─ Baselines established

Phase 2: Detection & Analysis
├─ Monitor for attacks
├─ Identify indicators
├─ Classify incidents
└─ Activate response

Phase 3: Containment
├─ Short-term containment
├─ Long-term containment
├─ Eradicate threat
└─ Preserve evidence

Phase 4: Eradication
├─ Remove malware
├─ Patch vulnerabilities
├─ Reset credentials
└─ Rebuild systems

Phase 5: Recovery
├─ Restore from backup
├─ Rebuild systems
├─ Validate security
└─ Restore services

Phase 6: Post-Incident
├─ Lessons learned
├─ Document findings
├─ Improve processes
└─ Update playbooks
```

---

## Part 1: Preparation Phase

### 1.1 Logging Infrastructure Setup

```bash
# Enable Apache access logging
LogLevel warn
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
CustomLog logs/access.log combined

# Enable MySQL general query log
SET GLOBAL general_log = 'ON';
SET GLOBAL log_output = 'TABLE';

# Enable system audit logging
auditctl -w /var/www/dvwa -p wa -k dvwa_changes
auditctl -w /etc/passwd -p wa -k passwd_changes
```

### 1.2 Monitoring Setup

```yaml
Alerts Configured:
  - Failed login attempts (5+ in 10 min)
  - SQL injection patterns
  - XSS payload detection
  - Privilege escalation attempts
  - Large data exfiltration (>100MB)
  - Unauthorized admin access
  - File modification alerts
  - Process execution monitoring
```

### 1.3 Mini SIEM Setup (ELK Stack)

**Installation:**
```bash
# Install Elasticsearch
curl -X PUT "localhost:9200/security-logs"

# Install Kibana
docker run -d -p 5601:5601 docker.elastic.co/kibana/kibana:7.14.0

# Install Logstash
filter {
  if [type] == "apache_access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
  if [message] =~ /SQL|UNION|DROP|INSERT/ {
    mutate { add_tag => [ "sql_injection_attempt" ] }
  }
  if [message] =~ /<script>|onerror=|onclick=/ {
    mutate { add_tag => [ "xss_attempt" ] }
  }
}
```

**Kibana Dashboards:**
```
Dashboard 1: Attack Detection
├─ Failed login attempts over time
├─ SQL injection pattern matches
├─ XSS payload detection
└─ Suspicious user activity

Dashboard 2: System Health
├─ CPU usage trends
├─ Memory consumption
├─ Disk I/O patterns
└─ Network traffic volume

Dashboard 3: Security Events
├─ File modifications
├─ Process executions
├─ User privilege changes
└─ Service restart attempts
```

---

## Part 2: Detection Phase

### 2.1 Indicators of Compromise (IOCs)

**Network IOCs:**
```
Protocol Anomalies:
- Unusual port usage
- DNS queries for known malicious domains
- C&C communication patterns
- Exfiltration attempts

Log Pattern IOCs:
- SQL injection payloads
  └─ UNION SELECT patterns
  └─ Time delay requests
  └─ Database enumeration queries

- XSS attempts
  └─ Script tag injection
  └─ Event handler injection
  └─ Encoded payloads

- Brute force attacks
  └─ Multiple failed logins
  └─ Progressive password attempts
  └─ Dictionary attacks
```

**File IOCs:**
```
Created Files:
- Suspicious shell scripts in /tmp
- Web shells in document root
- Config files with hardcoded credentials

Modified Files:
- Application source code changes
- Configuration file alterations
- Permission changes
- Timestamp inconsistencies
```

**Process IOCs:**
```
Suspicious Processes:
- Child processes of web server (apache2)
- Network connections from PHP processes
- File access patterns
- Resource consumption spikes
```

### 2.2 Monitoring Rules

```
Rule 1: SQL Injection Detection
├─ Trigger: Pattern match "' OR '1'='1"
├─ Severity: CRITICAL
├─ Action: Generate alert, log payload
└─ Response: Investigate immediately

Rule 2: XSS Attack Detection
├─ Trigger: Pattern match "<script>" in user input
├─ Severity: HIGH
├─ Action: Generate alert, capture context
└─ Response: Review next available

Rule 3: Brute Force Detection
├─ Trigger: 5+ failed logins in 10 minutes
├─ Severity: HIGH
├─ Action: Block source IP, alert SOC
└─ Response: Investigate source

Rule 4: Data Exfiltration
├─ Trigger: Transfer >100MB in 1 hour
├─ Severity: CRITICAL
├─ Action: Block connection, preserve evidence
└─ Response: Incident response activation

Rule 5: Privilege Escalation
├─ Trigger: UID change from <500 to 0
├─ Severity: CRITICAL
├─ Action: Immediate alert, session capture
└─ Response: Isolate system immediately
```

---

## Part 3: Detection & Analysis

### 3.1 Attack Detection Walkthrough

**Scenario: SQL Injection Attack on DVWA**

**Time: 14:00 UTC - Attack Initiated**

```
Attacker Action:
$ curl "http://192.168.56.102/dvwa/vulnerabilities/sqli/?id=1' OR '1'='1"

System Detection:
1. Apache access log entry
   [05/Oct/2026 14:10:23] "GET /dvwa/vulnerabilities/sqli/?id=1' OR '1'='1"
   └─ Severity flag: ALERT
   └─ Pattern matched: SQL_INJECTION_ATTEMPT

2. Elasticsearch indexed
   └─ Alert generated in Kibana dashboard
   └─ Threshold exceeded: SQL injection pattern

3. Automated Response
   └─ Email alert sent to SOC
   └─ Slack notification posted
   └─ Log exported for analysis
```

**Time: 14:15 UTC - Scope Analysis**

```
Alert Details:
├─ Source IP: 192.168.56.101
├─ Target: /dvwa/vulnerabilities/sqli/
├─ Payload: 1' OR '1'='1
├─ Parameter: id
└─ Timestamp: 14:10:23 UTC

Initial Questions:
Q: Is this a test?           A: Unknown - Investigate
Q: Is this internal?          A: Yes (192.168.56.x)
Q: Is system compromised?     A: Unknown - Analyze
Q: What data accessed?        A: Requires log analysis
Q: Was it successful?         A: Requires DB log check

Analysis Results:
├─ Request returned 200 OK (Success indicator)
├─ Response size: 4.2MB (Large result set)
├─ Database query time: 0.5s (Normal)
└─ Multiple user records in response

Classification: CONFIRMED SQL INJECTION ATTACK
```

---

## Part 4: Containment Phase

### 4.1 Short-term Containment

**Immediate Actions (First 5 Minutes):**

```
Action 1: Identify Attack Source
├─ IP: 192.168.56.101 (Attacker Kali Linux)
├─ Port: 52341 (Ephemeral)
├─ Protocol: HTTP/1.1
├─ User-Agent: curl/7.68.0
└─ Timestamp: 14:10:23 UTC

Action 2: Block Access
├─ Firewall rule added
├─ iptables -I INPUT -s 192.168.56.101 -j DROP
├─ WAF rule enabled
├─ Rate limiting: 10 requests/min
└─ Verification: Ping blocked ✓

Action 3: Alert Team
├─ Email to Security Team: "CRITICAL ALERT"
├─ Page on-call responder
├─ Post to incident Slack channel
├─ Document timeline
└─ Initiate incident #INC-2026-10-001

Action 4: Preserve Evidence
├─ Backup access logs
├─ Export database query logs
├─ Capture system state (ps, netstat)
├─ Screenshot dashboards
└─ Export IDS alerts
```

**Timeline: T+0 to T+30 minutes**

### 4.2 Long-term Containment

**Extended Actions (30 minutes - 2 hours):**

```
Action 1: Isolate Affected Systems
├─ Move DVWA to isolated network segment
├─ Disable external connectivity
├─ Maintain forensic access
├─ Block database replication

Action 2: Revoke Access
├─ Force logout all users
├─ Disable compromised admin account
├─ Revoke API tokens
├─ Terminate SSH sessions

Action 3: Patch Vulnerabilities
├─ Deploy SQL injection fix
├─ Add input validation
├─ Implement prepared statements
├─ Configure WAF rules

Action 4: Monitor for Persistence
├─ Check for backdoors
├─ Search for web shells
├─ Review cron jobs
├─ Examine startup scripts
└─ Monitor for re-exploitation attempts

Action 5: Prepare Recovery
├─ Identify clean backup
├─ Test restore procedure
├─ Prepare patched image
├─ Stage for deployment
```

---

## Part 5: Eradication Phase

### 5.1 Malware/Backdoor Removal

```bash
# Check for web shells
find /var/www -name "*.php" -newer /var/www/index.php
find /var/www -type f -size 0 -o -size +10000k

# Remove suspicious files
rm -f /var/www/shell.php
rm -f /tmp/backdoor.sh

# Check for cron backdoors
cat /etc/crontab
ls -la /etc/cron.d/

# Audit system binaries
md5sum /bin/* | grep -v "OK" | cut -d' ' -f3
```

### 5.2 Vulnerability Patching

```php
// Patch 1: SQL Injection Fix
$query = "SELECT * FROM users WHERE id = ?";
$stmt = $pdo->prepare($query);
$stmt->execute([$_GET['id']]);

// Patch 2: XSS Prevention
echo htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');

// Patch 3: CSRF Protection
if ($_POST['csrf_token'] !== $_SESSION['csrf_token']) {
    die('Attack detected');
}

// Patch 4: Rate Limiting
if ($attempts > 5 && time() - $last_attempt < 600) {
    header('HTTP/1.1 429 Too Many Requests');
    die();
}
```

### 5.3 Credential Reset

```bash
# MySQL
mysql -u root -e "SET PASSWORD FOR 'dvwa'@'localhost' = PASSWORD('NewSecurePass123!');"

# Application users
UPDATE users SET password = SHA2('NewPassword123!', 256) WHERE id > 0;

# System user
passwd admin

# Web server
sudo systemctl restart apache2
```

---

## Part 6: Recovery Phase

### 6.1 System Restoration

```bash
# Restore from clean backup
mysql < backup_2026-10-04.sql

# Verify integrity
sha256sum /var/www/dvwa/* > /tmp/current_hashes.txt
diff /tmp/original_hashes.txt /tmp/current_hashes.txt

# Apply security updates
apt-get update && apt-get upgrade -y

# Restart services
systemctl restart apache2
systemctl restart mysql
```

### 6.2 Validation Testing

```bash
# Security test 1: SQL Injection
curl "http://localhost/dvwa/vulnerabilities/sqli/?id=1' OR '1'='1"
# Expected: Error or no change in results

# Security test 2: XSS
curl "http://localhost/dvwa/guestbook/" -d "message=<script>alert('xss')</script>"
# Expected: Script encoded, not executed

# Security test 3: CSRF
# Check for CSRF token in forms
grep -r "csrf_token" /var/www/dvwa/

# Load test
ab -n 1000 -c 10 http://localhost/dvwa/
```

### 6.3 System Reactivation

```
Phase 1: Pre-production Testing (2 hours)
├─ Full security scan
├─ Functionality verification
├─ Performance baseline
└─ Staff sign-off required

Phase 2: Staged Production Rollout (4 hours)
├─ Hour 1: Internal users only (5% traffic)
├─ Hour 2: Limited external access (25% traffic)
├─ Hour 3: Monitored full rollout (75% traffic)
└─ Hour 4: Full production (100% traffic)

Phase 3: Post-Recovery Monitoring (24-48 hours)
├─ Real-time monitoring
├─ Alert threshold adjustments
├─ Performance monitoring
└─ Daily status reports
```

---

## Part 7: Post-Incident Phase

### 7.1 Lessons Learned

**What Went Wrong:**
1. ✗ SQL injection vulnerability in code
2. ✗ No input validation
3. ✗ No prepared statements
4. ✗ No security headers
5. ✗ Weak password policy

**What Went Right:**
1. ✓ Detection happened within 45 minutes
2. ✓ Incident response plan worked
3. ✓ Team coordinated effectively
4. ✓ Evidence preserved properly
5. ✓ Recovery successful in 8 hours

### 7.2 Improvements Implemented

```yaml
Improvements:
  Code Review:
    - Implement peer review process
    - Security code review checklist
    - Automated vulnerability scanning

  Testing:
    - Unit tests for security
    - SAST integration in CI/CD
    - DAST penetration testing
    - Fuzzing tests

  Monitoring:
    - Real-time alerting
    - ELK stack dashboards
    - Email notifications
    - Slack integration

  Training:
    - Secure coding course
    - OWASP Top 10 training
    - Incident response drills
    - Security awareness updates

  Processes:
    - Updated playbooks
    - Better escalation paths
    - Clearer communication
    - Quarterly drills
```

---

## Incident Response Checklist

```
Preparation Phase
[ ] Monitoring tools deployed
[ ] Alert rules configured
[ ] Response procedures documented
[ ] Team trained and identified
[ ] Contact list updated
[ ] Backup procedures tested

Detection Phase
[ ] Alert received and logged
[ ] Initial triage completed
[ ] Severity assessed
[ ] Scope determined
[ ] Evidence preserved

Containment Phase
[ ] Attacker blocked/isolated
[ ] Affected systems isolated
[ ] Credentials invalidated
[ ] Access logs exported
[ ] Team notified

Eradication Phase
[ ] Malware removed
[ ] Vulnerabilities patched
[ ] Credentials reset
[ ] Systems hardened
[ ] Backdoors checked

Recovery Phase
[ ] Clean backup restored
[ ] System integrity verified
[ ] Security testing passed
[ ] Systems brought online
[ ] Monitoring activated

Post-Incident Phase
[ ] Timeline documented
[ ] Root cause identified
[ ] Improvements planned
[ ] Lessons shared
[ ] Follow-up testing scheduled
```

---

**Status:** ✅ COMPLETE  
**Incident Response Plan:** OPERATIONAL  
**Team Readiness:** TRAINED
