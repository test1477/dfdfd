To update the `jfrog_report.py` script and integrate it into your GitHub Actions workflow, we will make the following changes:

1. **Handle Multiple Organizations:**
   - Update the script to accept the organization (`--org`) as a command-line argument.
   - Fetch the JFrog URL and API key dynamically based on the organization name.

2. **Adapt to the GitHub Actions Workflow:**
   - The workflow will pass the environment variables for the API key and JFrog URL, so the script will use these to make API requests.

Here is the updated `jfrog_report.py` script that supports multiple organizations and accepts command-line arguments for the organization, JFrog URL, and output directory.

### Updated `jfrog_report.py` Script:

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
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    """
    Recursively fetches all artifacts in the specified repository.
    """
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

# Function to process repository artifacts without extra API calls
def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository using batch metadata from the recursive listing.
    """
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

### Workflow (`.github/workflows/generate_jfrog_reports.yml`):

```yaml
name: Generate JFrog Reports

on:
  schedule:
    - cron: "0 9 * * *"  # Runs daily at 9 AM UTC
  workflow_dispatch:  # Allows manual triggering

jobs:
  generate-report:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set Up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.9"

      - name: Install Dependencies
        run: pip install requests

      - name: Run Report Script for EV
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
        run: |
          python jfrog_report.py --org ev --jfrog-url "https://frigate.jfrog.io/artifactory" --output reports/ev

      - name: Run Report Script for PPA
        env:
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          python jfrog_report.py --org ppa --jfrog-url "https://ppa.jfrog.io/artifactory" --output reports/ppa

      - name: Upload Reports to JFrog (EV)
        run: |
          curl -H "X-JFrog-Art-Api:${{ secrets.JFROG_API_KEY_EV }}" -T reports/ev/EV_EOL_$(date +%F).csv "https://frigate.io/artifactory/reports/ev/EV_EOL_$(date +%F).csv"

      - name: Upload Reports to JFrog (PPA)
        run: |
          curl -H "X-JFrog-Art-Api:${{ secrets.JFROG_API_KEY_PPA }}" -T reports/ppa/EV_EOL_$(date +%F).csv "https://ppa.jfrog.io/artifactory/reports/ppa/EV_EOL_$(date +%F).csv"
```

### Key Updates:
1. **Python Script:**
   - The `jfrog_report.py` script now accepts command-line arguments for the organization (`--org`), JFrog URL (`--jfrog-url`), and output directory (`--output`).
   - It dynamically fetches the API key based on the organization (`JFROG_API_KEY_EV` or `JFROG_API_KEY_PPA`) using `os.getenv()`.

2. **GitHub Actions Workflow:**
   - The workflow triggers the report generation for both `ev` and `ppa` organizations.
   - It uploads the generated reports to the respective JFrog Artifactory locations.

### Next Steps:
1. **Add API Keys:** Ensure you have the JFrog API keys (`JFROG_API_KEY_EV` and `JFROG_API_KEY_PPA`) stored as GitHub Secrets.
2. **Push the Changes:** Commit and push the updated Python script and workflow to your GitHub repository.
3. **Run the Workflow:** The workflow will run automatically based on the cron schedule or can be triggered manually.

Let me know if you need any further adjustments!
