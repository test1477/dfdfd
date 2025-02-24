The **`KeyError: 'html_url'`** and the **`403 Forbidden` error** suggest a few potential issues:

1. **Permission Issues (403 Forbidden)**  
   - Your GitHub token might not have the correct permissions (e.g., `repo`, `security_events` scope).
   - If your token has expired, it could cause **403 errors**.
   - API rate limits might be blocking your requests.

2. **Missing 'html_url' Key in API Response**  
   - If a repository is private or deleted, the API might not return the `html_url` key.
   - The `getDependabotAlertsRepo()` function may need to handle missing keys more gracefully.

---

### **🔧 Fixes & Enhancements**
✅ **Handle Missing Keys with `.get()`**  
✅ **Check for API Rate Limits & Authentication Issues**  
✅ **Improve Error Handling for 403 Errors**

Here’s an **updated version** of the script with better error handling:

---

### ** Updated Script**
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
    
    if res.status_code == 403:
        print(f" ERROR: Access Denied (403) - Check token permissions for team {team}")
        return []
    
    if res.status_code != 200:
        print(f" ERROR: Failed to fetch repos for team {team}: {res.status_code}")
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

    if res.status_code == 403:
        print(f" ERROR: Access Denied (403) - Check token permissions for repo {repo}")
        return []
    
    if res.status_code != 200:
        print(f" ERROR: Failed to fetch alerts for repo {repo}: {res.status_code}")
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
        print(f" Fetching repositories for team: {team}")
        team_repos = getTeamRepos(team, token)

        for repo in team_repos:
            repo_name = repo.get('name', 'Unknown')
            print(f" Processing repository: {repo_name}")
            alerts = getDependabotAlertsRepo(repo_name, token)
            if alerts:
                print(f"{repo_name} has {len(alerts)} open Dependabot alerts!")
                for alert in alerts:
                    alert['repository'] = {"full_name": repo_name}  # Ensure full_name exists
                all_alerts += alerts

    return all_alerts

# Main function
def main():
    # Get GitHub Token from environment variable
    GHToken = os.getenv("ACCESS_TOKEN")
    
    if not GHToken:
        print(" ERROR: GitHub Token is missing! Set ACCESS_TOKEN environment variable.")
        return

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
                vuln_id = alert.get('security_advisory', {}).get('cve_id') or alert.get('security_advisory', {}).get('ghsa_id')

                # Extract CVSS Version
                cvss_version_match = re.search(r'CVSS:(.*?)\/', str(alert.get('security_advisory', {}).get('cvss', {}).get('vector_string', '')))
                cvss_version = cvss_version_match.group(1) if cvss_version_match else "N/A"

                # Unique ID format: GHASID-<alert_number>_<package_name>_<repo_name>
                unique_id = f"GHASID-{alert.get('number', 'Unknown')}_{alert.get('dependency', {}).get('package', {}).get('name', 'Unknown')}_{alert.get('repository', {}).get('full_name', 'Unknown').replace('/', '')}"

                # GitHub URL (Handle missing keys)
                github_url = alert.get('repository', {}).get('html_url', 'N/A')

                # Write row to CSV
                writer.writerow([
                    alert.get('repository', {}).get('full_name', 'Unknown'),
                    f"GHASID-{alert.get('number', 'Unknown')}",
                    alert.get('dependency', {}).get('package', {}).get('name', 'Unknown'),
                    alert.get('dependency', {}).get('package', {}).get('ecosystem', 'Unknown'),
                    alert.get('dependency', {}).get('manifest_path', 'Unknown'),
                    alert.get('security_advisory', {}).get('severity', 'Unknown'),
                    alert.get('security_advisory', {}).get('summary', 'Unknown').replace('\n', ' ').replace('\r', ''),
                    alert.get('security_advisory', {}).get('description', 'Unknown').replace('\n', ' ').replace('\r', ''),
                    unique_id,
                    alert.get('security_vulnerability', {}).get('first_patched_version', {}).get('identifier', 'Not patched'),
                    alert.get('security_advisory', {}).get('cvss', {}).get('score', 'N/A'),
                    cvss_version,
                    str(alert.get('security_advisory', {}).get('vulnerabilities', 'N/A')),
                    alert.get('security_vulnerability', {}).get('vulnerable_version_range', 'Unknown'),
                    github_url,
                    alert.get('created_at', 'Unknown')
                ])

    print(f"✅ CSV file saved successfully: {OUTPUT_FILE}")

# Run the script
if __name__ == "__main__":
    main()
```

---

### **🔹 Fixes & Enhancements**
✅ **Prevents `KeyError: 'html_url'`**  
- Uses `.get()` to avoid missing keys.

✅ **Handles `403 Forbidden` Errors**  
- If GitHub denies access, an error message is printed.

✅ **Checks for Missing `ACCESS_TOKEN`**  
- If the token is missing, the script stops gracefully.

✅ **Prevents Crashes on Missing Data**  
- Default values like `"Unknown"` are used instead of crashing.

---

### **🚀 Try This**
1. **Check Your GitHub Token:**  
   - Ensure it has **`repo`** and **`security_events`** permissions.
   - Generate a new **Personal Access Token (PAT)** if needed.

2. **Run the Script Again:**  
   ```
   python script.py
   ```

This should now **handle missing fields, API errors, and token issues.** Let me know if you need more fixes! 🚀
