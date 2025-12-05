# Snyk - Exporting & Exploiting Results

---

## Exporting Results

> **Important:** All export commands must be run from your project directory (e.g., `keiken-dragon-ai`)
> Results will be saved in `../Scanner/Snyk/Results/` directory

### Method 1: JSON Export (synk-setup-scan.md)
Generate JSON reports for automated processing:

### Method 2: SARIF Export (for CI/CD integration)
```bash
snyk test --sarif-file-output=../Scanner/Snyk/Results/snyk-results.sarif
```

### Method 3: HTML Report
```bash
snyk test --all-projects > ../Scanner/Snyk/Results/snyk-report.html
```

### Method 4: Snyk Web Dashboard
Send results to Snyk cloud dashboard for continuous monitoring:
```bash
snyk monitor --all-projects
```
Access results at: https://app.snyk.io/
---

## Understanding & Exploiting Results

### SCA Results Structure (Dependencies)

```json
{
  "vulnerabilities": [
    {
      "id": "SNYK-JS-LODASH-590103",
      "title": "Prototype Pollution",
      "severity": "high",
      "packageName": "lodash",
      "version": "4.17.15",
      "fixedIn": ["4.17.21"],
      "cvssScore": 7.4,
      "CVSSv3": "CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:N",
      "exploit": "Proof of Concept",
      "isPatchable": true,
      "upgradePath": ["lodash@4.17.21"]
    }
  ],
  "summary": {
    "critical": 2,
    "high": 15,
    "medium": 45,
    "low": 12
  }
}
```

**Key Fields to Review:**
- `severity`: Prioritize Critical > High > Medium > Low
- `fixedIn`: Target version to upgrade to
- `exploit`: Maturity level (No Known Exploit, Proof of Concept, Functional)
- `isPatchable`: Can be fixed by upgrading dependency
- `upgradePath`: Exact upgrade command/version needed

---

### SAST Results Structure (Code Analysis)

```json
{
  "runs": [{
    "results": [
      {
        "ruleId": "python/SqlInjection",
        "level": "error",
        "message": {
          "text": "Unsanitized input from request is used in SQL query"
        },
        "locations": [{
          "physicalLocation": {
            "artifactLocation": {
              "uri": "backend/api/users.py"
            },
            "region": {
              "startLine": 45,
              "startColumn": 12
            }
          }
        }],
        "cwe": ["CWE-89"],
        "priority": "high"
      }
    ]
  }]
}
```

**Key Fields to Review:**
- `ruleId`: Type of security issue
- `level`: error (high), warning (medium), note (low)
- `locations`: Exact file and line number
- `cwe`: Common Weakness Enumeration reference
- `message.text`: Description of the vulnerability

---

## Prioritization Strategy

### 1. Critical/High Severity with Known Exploits
**Priority: IMMEDIATE**
```bash
# Filter critical issues with exploits
cat ../Scanner/Snyk/Results/snyk-sca-results.json | jq '.vulnerabilities[] | select(.severity == "critical" and .exploit != "Not Defined")'
```

### 2. Patchable Vulnerabilities
**Priority: HIGH**
```bash
# Filter patchable issues
cat ../Scanner/Snyk/Results/snyk-sca-results.json | jq '.vulnerabilities[] | select(.isPatchable == true)'
```

### 3. SAST Errors (Code-level security issues)
**Priority: HIGH**
```bash
# Filter SAST errors
cat ../Scanner/Snyk/Results/snyk-sast-results.json | jq '.runs[].results[] | select(.level == "error")'
```

### 4. Medium/Low or No Known Exploit
**Priority: MEDIUM**
- Plan remediation in next sprint
- Monitor for exploit developments

---

## Generating Summary Reports

### Vulnerability Count by Severity
```bash
# SCA summary
cat ../Scanner/Snyk/Results/snyk-sca-results.json | jq '.summary'

# SAST summary by level
cat ../Scanner/Snyk/Results/snyk-sast-results.json | jq '[.runs[].results[] | .level] | group_by(.) | map({level: .[0], count: length})'
```

### Top 10 Most Critical Packages
```bash
cat ../Scanner/Snyk/Results/snyk-sca-results.json | jq '[.vulnerabilities[] | select(.severity == "critical" or .severity == "high")] | group_by(.packageName) | map({package: .[0].packageName, count: length}) | sort_by(.count) | reverse | .[0:10]'
```

### Files with Most SAST Issues
```bash
cat ../Scanner/Snyk/Results/snyk-sast-results.json | jq '[.runs[].results[] | .locations[].physicalLocation.artifactLocation.uri] | group_by(.) | map({file: .[0], count: length}) | sort_by(.count) | reverse | .[0:10]'
```

---

## Continuous Monitoring

### Track Trends Over Time
```bash
# Monitor project in Snyk dashboard
snyk monitor --all-projects --project-name="dragon-develop"
```

### Set Up Alerts
- Configure Snyk webhooks for new vulnerabilities
- Enable email alerts for critical issues
- Integrate with Slack/Teams for notifications

---

## Action Plan Template

1. **Immediate Actions (Critical/High with Exploits)**
   - [ ] Fix SQL Injection in `backend/api/users.py:45`
   - [ ] Upgrade lodash to 4.17.21
   - [ ] Patch critical OS vulnerabilities in base image

2. **Short-term (Patchable High/Medium)**
   - [ ] Update all dependencies with available fixes
   - [ ] Refactor code with SAST warnings

3. **Long-term (Low Severity / No Fix Available)**
   - [ ] Monitor for patches
   - [ ] Consider alternative libraries
   - [ ] Implement workarounds/mitigations

---

## Integration Tips

### CI/CD Integration
```bash
# Fail build on critical vulnerabilities
snyk test --severity-threshold=high
```

### Git Pre-commit Hook
```bash
#!/bin/bash
snyk test || exit 1
```
---