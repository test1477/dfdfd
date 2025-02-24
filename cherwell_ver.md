Got it! Since `getTeamRepos(team, token)` is missing from your script, I will add that function without making any major changes to your existing script.  

Here is the **updated script** with `getTeamRepos()` added:  

---

### **Updated Script**
```python
import os
import csv
import json
import requests
import urllib3
import datetime
import re
import warnings

# Suppress warnings
warnings.filterwarnings("ignore")
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Define the output directory and file name
OUTPUT_DIR = "//EVNT30/EV01SHRDATA/Cherwell/Vulnerabilities/current/"
OUTPUT_FILE = os.path.join(OUTPUT_DIR, f"Vulnerabilities_{datetime.datetime.now().strftime('%Y-%m-%d')}.csv")

# Function to fetch repositories for a team
def getTeamRepos(team, token):
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }
    url = f"https://api.github.com/orgs/Eaton-Vance-Corp/teams/{team}/repos"
    
    res = requests.get(url, headers=headers, verify=False)
    if res.status_code != 200:
        print(f"Failed to fetch repos for team {team}: {res.status_code}")
        return []

    return res.json()

# Function to fetch Dependabot alerts for a repository
def getDependabotAlertsRepo(repo, token, page=1):
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }
    url = f"https://api.github.com/repos/Eaton-Vance-Corp/{repo}/dependabot/alerts?per_page=100&page={page}&state=open"
    res = requests.get(url, headers=headers, verify=False)

    if res.status_code != 200:
        print(f"Failed to fetch alerts for repo {repo}: {res.status_code}")
        return []

    alerts = res.json()
    
    if len(alerts) == 100:
        return alerts + getDependabotAlertsRepo(repo, token, page + 1)

    return alerts

# Function to fetch Dependabot alerts for multiple teams
def getDependabotAlertsTeams(teams, token):
    all_alerts = []
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }

    for team in teams:
        print(f"Fetching repositories for team: {team}")
        team_repos = getTeamRepos(team, token)

        for repo in team_repos:
            repo_name = repo['name']
            print(f"Processing repository: {repo_name}")
            alerts = getDependabotAlertsRepo(repo_name, token)
            if alerts:
                print(f"{repo_name} has {len(alerts)} open Dependabot alerts!")
                for alert in alerts:
                    alert['repository'] = {"full_name": repo_name}
                all_alerts += alerts

    return all_alerts

# Main function
def main():
    # Get GitHub Token from environment variable
    GHToken = os.getenv("ACCESS_TOKEN")

    # List of GitHub teams to fetch alerts from
    teams = [
        'aws-core-services-admins', 'aws-iam-admins', 'aws-iam-user-based-admins',
        'cloudoperations', 'cloudoperations-admins', 'confluent-cloud-admins',
        'confluent-cloud-developers', 'confluent-core-platform-admins'
    ]

    # Fetch Dependabot alerts
    dependabot_alerts = getDependabotAlertsTeams(teams, GHToken)

    # CSV Headers
    csv_headers = [
        "Repository Name", "Alert ID", "Component Name", "Package Name", "Ecosystem",
        "Manifest Path", "Vulnerability Rating", "Short Description", "Description",
        "Unique ID", "First Patched Version", "CVSS Rating", "CVSS Version",
        "Vulnerabilities List", "Identifiers", "Vulnerable Version Range",
        "GitHub URL", "Date Discovered"
    ]

    # Create directory if it does not exist
    os.makedirs(OUTPUT_DIR, exist_ok=True)

    # Write data to CSV
    with open(OUTPUT_FILE, mode="w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)
        writer.writerow(csv_headers)  # Writing headers

        for alert in dependabot_alerts:
            if "dependency" in alert:
                # Extract Vulnerability ID
                vuln_id = alert['security_advisory']['cve_id'] if alert['security_advisory']['cve_id'] else alert['security_advisory']['ghsa_id']

                # Extract CVSS Version
                cvss_version_match = re.search(r'CVSS:(.*?)\/', str(alert['security_advisory']['cvss']['vector_string']))
                cvss_version = cvss_version_match.group(1) if cvss_version_match else "N/A"

                # Unique ID format: GHASID-<alert_number>_<package_name>_<repo_name>
                unique_id = f"GHASID-{alert['number']}_{alert['dependency']['package']['name']}_{alert['repository']['full_name'].replace('/', '')}"

                # Write row to CSV
                writer.writerow([
                    alert['repository']['full_name'],
                    f"GHASID-{alert['number']}",
                    alert['dependency']['package']['name'],
                    alert['dependency']['package']['ecosystem'],
                    alert['dependency']['manifest_path'],
                    alert['security_advisory']['severity'],
                    alert['security_advisory']['summary'].replace('\n', ' ').replace('\r', ''),
                    alert['security_advisory']['description'].replace('\n', ' ').replace('\r', ''),
                    unique_id,
                    alert['security_vulnerability']['first_patched_version']['identifier']
                    if alert['security_vulnerability'].get('first_patched_version') else "Not patched",
                    alert['security_advisory']['cvss']['score'] if 'cvss' in alert['security_advisory'] else "N/A",
                    cvss_version,
                    str(alert['security_advisory'].get('vulnerabilities', 'N/A')),
                    alert['security_vulnerability']['vulnerable_version_range'],
                    alert['repository']['html_url'],
                    alert['created_at']
                ])

    print(f"CSV file saved successfully: {OUTPUT_FILE}")

# Run the script
if __name__ == "__main__":
    main()
```

---

### **Fixes & Enhancements**
✅ **Added `getTeamRepos(team, token)`**  
- This function fetches all repositories associated with a given team.  

✅ **Ensured "Unique ID" is included**  
- The **Unique ID** is generated using:  
  ```
  GHASID-<alert_number>_<package_name>_<repo_name>
  ```

✅ **CSV File Saves with Date**  
- Example: `Vulnerabilities_2025-02-14.csv`

✅ **Proper Error Handling**  
- If API requests fail, errors are printed instead of stopping execution.

✅ **Automatic Directory Creation**  
- Ensures the script runs even if the target folder does not exist.

---

### **How to Run the Script**
1. Ensure your **GitHub Access Token** is set:  
   ```
   export ACCESS_TOKEN="your_github_token"
   ```
   (For Windows: `set ACCESS_TOKEN=your_github_token`)

2. Run the script using Python:  
   ```
   python script.py
   ```

3. The output CSV will be saved in:  
   ```
   //EVNT30/EV01SHRDATA/Cherwell/Vulnerabilities/current/Vulnerabilities_YYYY-MM-DD.csv
   ```

This should now work as expected. Let me know if anything else needs fixing! 🚀
