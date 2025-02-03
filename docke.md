### **Updated Python Script: `report_script.py`**

I've added error handling to ensure the script continues even if one repository fails. It logs errors clearly without stopping the execution.  

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
    except requests.RequestException as e:
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
    except requests.RequestException as e:
        print(f"Failed to list artifacts for {repo_name}: {e}")
        return []

# Function to process repository artifacts
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
                package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
                version = segments[-2]
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
        return []  # Continue with other repositories

# Function to save data to CSV
def save_to_csv(data, filename):
    """
    Saves extracted metadata to a CSV file.
    """
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    
    os.makedirs(os.path.dirname(filename), exist_ok=True)  # Ensure output directory exists

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

    if not repositories:
        print("No repositories found or failed to fetch repositories.")
        return

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")

        print(f"Processing repository: {repo_name} (Type: {package_type})")

        try:
            # Process only Docker repositories
            if package_type == "Docker":
                repo_details = process_repository(repo_name, package_type)
                all_repo_details.extend(repo_details)
        except Exception as e:
            print(f"Skipping repository {repo_name} due to unexpected error: {e}")

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = os.path.join(args.output, f"EV_EOL_{current_date}_docker.csv")
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### **Key Updates**  

1. **Error Handling:**  
   - Wrapped API calls in `try-except` blocks to **prevent script termination** if one repository fails.  
   - Errors are logged clearly with repository-specific messages.

2. **Log Messages:**  
   - Added logs to **track the progress** of each repository being processed.  
   - Displays errors if repositories or artifacts cannot be fetched.

3. **Directory Handling:**  
   - Automatically **creates the output directory** if it doesn’t exist (`os.makedirs`).

4. **Resilient to Failures:**  
   - If any repo fails, the script continues with others.  
   - Repositories with no data will be skipped gracefully.

---



Here's the updated **GitHub Actions workflow** (`.github/workflows/jfrog_report.yml`) with your specified schedule (`9 AM UTC` daily) and adjustments for **error handling, logging, and ensuring proper upload of reports**.

---

### **Updated GitHub Actions Workflow**
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
          echo "Generating report for EV..."
          python report_script.py --org ev --jfrog-url "https://frigate.io/artifactory" --output reports/ev || echo "EV report generation failed"

      - name: Run Report Script for PPA
        env:
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          echo "Generating report for PPA..."
          python report_script.py --org ppa --jfrog-url "https://ppa.jfrog.io/artifactory" --output reports/ppa || echo "PPA report generation failed"

      - name: Validate Reports Before Upload
        run: |
          ls -lh reports/ev || echo "EV report missing"
          ls -lh reports/ppa || echo "PPA report missing"

      - name: Upload Reports to JFrog (EV)
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
        run: |
          if [ -f "reports/ev/EV_EOL_$(date +%F).csv" ]; then
            echo "Uploading EV report..."
            curl -H "X-JFrog-Art-Api:${JFROG_API_KEY_EV}" -T reports/ev/EV_EOL_$(date +%F).csv "https://frigate.io/artifactory/reports/ev/EV_EOL_$(date +%F).csv" || echo "EV upload failed"
          else
            echo "EV report file not found, skipping upload."
          fi

      - name: Upload Reports to JFrog (PPA)
        env:
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          if [ -f "reports/ppa/EV_EOL_$(date +%F).csv" ]; then
            echo "Uploading PPA report..."
            curl -H "X-JFrog-Art-Api:${JFROG_API_KEY_PPA}" -T reports/ppa/EV_EOL_$(date +%F).csv "https://ppa.jfrog.io/artifactory/reports/ppa/EV_EOL_$(date +%F).csv" || echo "PPA upload failed"
          else
            echo "PPA report file not found, skipping upload."
          fi
```

---

### **Enhancements in This Update**
✔ **Improved Error Handling:**  
  - The script **does not fail the entire workflow** if one report fails.  
  - Uses `|| echo "Failure message"` to log errors instead of stopping execution.  

✔ **Logging & Validation:**  
  - `echo "Generating report..."` logs execution steps.  
  - **List files (`ls -lh reports/...`) before upload** to verify the output exists.  
  - Skips upload if the file is missing, preventing unnecessary API calls.  

✔ **Separate Environment Variables for Security:**  
  - `JFROG_API_KEY_EV` and `JFROG_API_KEY_PPA` are used **separately** in the appropriate steps.  

---

### **Next Steps**
1. **Ensure GitHub Secrets**:  
   - Set `JFROG_API_KEY_EV` and `JFROG_API_KEY_PPA` in GitHub Actions secrets.  

2. **Ensure `report_script.py` Works as Expected**  
   - Run it locally:  
     ```sh
     python report_script.py --org ev --jfrog-url "https://frigate.io/artifactory" --output reports/ev
     ```

3. **Commit & Push the Workflow File**  
   ```sh
   git add .github/workflows/jfrog_report.yml
   git commit -m "Update JFrog report workflow with error handling"
   git push
   ```

This will automatically trigger the workflow **daily at 9 AM UTC** or when manually executed.

Let me know if you need more refinements! 🚀

### **Sample Command to Run:**  
```bash
python report_script.py --org ev --jfrog-url "https://frigate.io/artifactory" --output reports/ev
```

Let me know if you need further changes.
