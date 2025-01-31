Here’s the full updated script with error handling to ensure the script continues execution even if a repository fails. Additionally, I've included a GitHub Actions workflow to automate this script.

---

### **Updated Python Script (`jfrog_report.py`)**
```python
import os
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime
import argparse

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# Parse command-line arguments
parser = argparse.ArgumentParser(description="Generate JFrog reports for multiple organizations.")
parser.add_argument('--org', required=True, help="The organization name (e.g., 'ev' or 'ppa').")
parser.add_argument('--jfrog-url', required=True, help="The JFrog Artifactory URL.")
parser.add_argument('--output', required=True, help="The output directory for the generated report.")
args = parser.parse_args()

# Get JFrog Artifactory instance details
JFROG_URL = args.jfrog_url
ARTIFACTORY_TOKEN = os.getenv(f'JFROG_API_KEY_{args.org.upper()}')  # Get API key for the given organization

if not ARTIFACTORY_TOKEN:
    print(f"Error: JFROG_API_KEY_{args.org.upper()} is not set in environment variables.")
    exit(1)

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

# Function to fetch all repositories
def fetch_repositories():
    """
    Fetches a list of all repositories in the Artifactory instance.
    """
    url = f"{JFROG_URL}/api/repositories"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Failed to fetch repositories: {e}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    """
    Recursively fetches all artifacts in the specified repository.
    """
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.exceptions.RequestException as e:
        print(f"Failed to list artifacts for {repo_name}: {e}")
        return []

# Function to process repository artifacts without extra API calls
def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository using batch metadata from the recursive listing.
    """
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "")
            artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
            created_date = artifact.get("lastModified", "")

            # Parse package name and version from artifact path
            segments = artifact_path.strip("/").split("/")
            if len(segments) >= 2:
                package_name = f"{repo_name}/{'/'.join(segments[:-2])}"  # Up to the version folder
                version = segments[-2]  # Second-to-last segment
            else:
                package_name = repo_name
                version = ""

            # Append details only for Docker repositories
            if package_type == "Docker":
                repo_details.append({
                    "repo_name": repo_name,
                    "package_type": package_type,
                    "package_name": package_name,
                    "version": version,
                    "url": artifact_url,
                    "created_date": created_date,
                    "license": "",  # Leave blank if not available
                    "secarch": "",  # Leave blank if not available
                    "artifactory_instance": "frigate.jfrog.io"
                })
        
        return repo_details
    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []

# Function to save data to CSV
def save_to_csv(data, filename):
    """
    Saves extracted metadata to a CSV file.
    """
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

# Main function to process all repositories
def main():
    """
    Main function to process all repositories and extract metadata.
    """
    repositories = fetch_repositories()
    all_repo_details = []

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")  # Dynamically fetch the package type
        print(f"Processing repository: {repo_name} (Type: {package_type})")

        # Process only Docker repositories
        if package_type == "Docker":
            repo_details = process_repository(repo_name, package_type)
            all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = os.path.join(args.output, f"EV_EOL_{current_date}_docker.csv")  # Save only Docker reports
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### **GitHub Actions Workflow (`.github/workflows/jfrog_report.yml`)**
This workflow runs the script on a scheduled basis or when triggered manually.

```yaml
name: JFrog Report Generation

on:
  schedule:
    - cron: '0 2 * * *'  # Runs every day at 2 AM UTC
  workflow_dispatch:  # Allows manual trigger

jobs:
  generate-report:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install Dependencies
        run: pip install requests

      - name: Run JFrog Report Script
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          python jfrog_report.py --org ev --jfrog-url "https://frigate.jfrog.io" --output "./reports"
          
      - name: Upload Report as Artifact
        uses: actions/upload-artifact@v3
        with:
          name: jfrog-report
          path: ./reports/*.csv
```

---

### **What’s New in This Update?**
✔ **Error Handling**: The script continues execution even if one repository fails. It skips failed repositories and prints a warning.  
✔ **Improved Logging**: More meaningful messages when failures occur.  
✔ **Automated Execution**: A GitHub Actions workflow is provided to run the script daily or manually.  
✔ **Secrets Management**: Uses GitHub secrets (`JFROG_API_KEY_EV`, `JFROG_API_KEY_PPA`) to securely store API keys.  
✔ **Artifact Storage**: The generated report is uploaded as an artifact in GitHub Actions.  

Would you like any additional refinements? 🚀
