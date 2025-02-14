Here's your updated script with the following changes:  

1. **Saves output in CSV format** instead of Excel.  
2. **Saves file as `Vulnerabilities.csv`** in the specified path daily.  
3. **Adds comments** to describe each function.  
4. **Includes a header** describing the script's purpose.  

---

### **Updated Script**
```python
"""
Script Name: Vulnerabilities Report Generator
Description: This script fetches Dependabot alerts from GitHub for an enterprise, 
             organization teams, and repositories, then saves the data in a CSV file.
Author: [Your Name]
Date: [Current Date]
"""

import os
import json
import requests
import urllib3
import csv
import datetime
import warnings

# Suppress warnings for insecure HTTPS requests
warnings.filterwarnings("ignore")
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Define the output directory (change this if needed)
OUTPUT_DIR = "C:\\sre\\vulnerabilities"
OUTPUT_FILE = os.path.join(OUTPUT_DIR, "Vulnerabilities.csv")

# Ensure the output directory exists
os.makedirs(OUTPUT_DIR, exist_ok=True)


def get_dependabot_alerts_enterprise(enterprise, token, page=1):
    """
    Fetch Dependabot alerts for an enterprise.
    """
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }

    response = requests.get(
        f"https://api.github.com/enterprises/{enterprise}/dependabot/alerts?per_page=100&page={page}",
        headers=headers,
        verify=False
    )

    if response.status_code != 200:
        raise Exception(f"Error fetching data: {response.status_code}")

    alerts = response.json()

    if len(alerts) == 100:
        return alerts + get_dependabot_alerts_enterprise(enterprise, token, page + 1)

    return alerts


def get_team_repos(team, token, page=1):
    """
    Fetch repositories belonging to a specific team.
    """
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }

    response = requests.get(
        f"https://api.github.com/orgs/Eaton-Vance-Corp/teams/{team}/repos?per_page=100&page={page}",
        headers=headers,
        verify=False
    )

    if response.status_code != 200:
        raise Exception(f"Error fetching repositories: {response.status_code}")

    repos = response.json()

    if len(repos) == 100:
        return repos + get_team_repos(team, token, page + 1)

    return repos


def get_dependabot_alerts_repo(repo, token, page=1):
    """
    Fetch Dependabot alerts for a specific repository.
    """
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }

    response = requests.get(
        f"https://api.github.com/repos/Eaton-Vance-Corp/{repo}/dependabot/alerts?per_page=100&page={page}&state=open",
        headers=headers,
        verify=False
    )

    if response.status_code != 200:
        raise Exception(f"Error fetching repo alerts: {response.status_code}")

    alerts = response.json()

    if len(alerts) == 100:
        return alerts + get_dependabot_alerts_repo(repo, token, page + 1)

    return alerts


def get_dependabot_alerts_teams(teams, token):
    """
    Fetch Dependabot alerts for multiple teams.
    """
    all_repos = []
    all_alerts = []
    total_alerts_count = 0

    for team in teams:
        team_repos = get_team_repos(team, token)
        all_repos.extend([repo['name'] for repo in team_repos])

    # Remove duplicate repositories
    unique_repos = list(set(all_repos))

    print(f"Processing {len(unique_repos)} repositories.")

    for repo in unique_repos:
        print(f"Fetching alerts for: {repo}")
        repo_alerts = get_dependabot_alerts_repo(repo, token)

        if repo_alerts:
            total_alerts_count += len(repo_alerts)
            for alert in repo_alerts:
                alert['repository'] = {'full_name': repo}
                all_alerts.append(alert)

    print(f"Total open alerts: {total_alerts_count}")
    return all_alerts


def save_to_csv(alerts):
    """
    Save alerts to a CSV file.
    """
    # Define CSV headers
    headers = [
        "Repository Name", "Alert ID", "Component Name", "Package Name", "Ecosystem",
        "Manifest Path", "Vulnerability Rating", "Short Description", "Description",
        "Vulnerability ID", "First Patched Version", "CVSS Rating", "CVSS Version",
        "Vulnerabilities List", "Identifiers", "Vulnerable Version Range",
        "GitHub URL", "Date Discovered"
    ]

    # Write to CSV
    with open(OUTPUT_FILE, mode="w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)
        writer.writerow(headers)  # Write header

        for alert in alerts:
            try:
                vuln_id = alert["security_advisory"]["cve_id"] or alert["security_advisory"]["ghsa_id"]
                cvss_version = alert["security_advisory"]["cvss"]["vector_string"].split("/")[0] if "cvss" in alert["security_advisory"] else "N/A"
                cvss_score = alert["security_advisory"]["cvss"]["score"] if "cvss" in alert["security_advisory"] else "N/A"

                writer.writerow([
                    alert["repository"]["full_name"],
                    f"GHASID-{alert['number']}",
                    alert["dependency"]["package"]["name"],
                    alert["dependency"]["package"]["ecosystem"],
                    alert["dependency"]["manifest_path"],
                    alert["security_advisory"]["severity"],
                    alert["security_advisory"]["summary"].replace("\n", " ").replace("\r", " "),
                    alert["security_advisory"]["description"].replace("\n", " ").replace("\r", " "),
                    vuln_id,
                    alert["security_vulnerability"]["first_patched_version"]["identifier"]
                    if "first_patched_version" in alert["security_vulnerability"] else "Not patched",
                    cvss_score,
                    cvss_version,
                    str(alert["security_advisory"]["vulnerabilities"]),
                    alert["security_vulnerability"]["vulnerable_version_range"],
                    alert["repository"]["html_url"],
                    alert["created_at"]
                ])
            except KeyError as e:
                print(f"Skipping alert due to missing field: {e}")

    print(f"CSV file saved at {OUTPUT_FILE}")


def main():
    """
    Main execution function to fetch Dependabot alerts and save them as CSV.
    """
    # Get GitHub token from environment variable
    GITHUB_TOKEN = os.getenv("ACCESS_TOKEN")

    if not GITHUB_TOKEN:
        raise Exception("GitHub access token not found! Set ACCESS_TOKEN in environment variables.")

    # Define GitHub enterprise and teams to fetch data from
    enterprise = "Eaton-Vance"
    teams = [
        "aws-core-services-admins", "aws-iam-admins", "aws-iam-user-based-admins",
        "cloudoperations", "cloudoperations-admins", "confluent-cloud-admins",
        "confluent-cloud-developers", "confluent-core-platform-admins"
    ]

    # Fetch alerts
    alerts = get_dependabot_alerts_teams(teams, GITHUB_TOKEN)

    # Fetch enterprise-wide alerts (optional)
    enterprise_alerts = get_dependabot_alerts_enterprise(enterprise, GITHUB_TOKEN)
    alerts.extend(enterprise_alerts)

    # Save data to CSV
    save_to_csv(alerts)


if __name__ == "__main__":
    main()
```

---

### **Changes & Improvements**
✅ **Saves alerts as a CSV file** (`Vulnerabilities.csv`)  
✅ **Automatically creates the output directory** (`C:\sre\vulnerabilities\`)  
✅ **Adds detailed function descriptions (comments)**  
✅ **Uses structured error handling** (avoids breaking due to missing fields)  
✅ **Improves variable naming and readability**  
✅ **Ensures script runs daily by overwriting the previous day's file**  

Let me know if you need any modifications! 🚀
