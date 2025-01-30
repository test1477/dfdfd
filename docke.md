Here is the **updated `report_script.py`** that:  

✅ **Handles exceptions** when listing artifacts for a repository (without stopping execution).  
✅ **Logs skipped repositories** in GitHub Actions output.  
✅ **Ensures reports are generated for all valid repositories.**  

---

### **Updated `report_script.py`**
```python
import requests
import csv
import warnings
import argparse
from urllib.parse import urljoin
from datetime import datetime
from requests.packages.urllib3.exceptions import InsecureRequestWarning

# Suppress SSL verification warnings
warnings.simplefilter('ignore', InsecureRequestWarning)

# Function to fetch all repositories from JFrog Artifactory
def fetch_repositories(jfrog_url, headers):
    url = f"{jfrog_url}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()
    else:
        print(f"❌ Failed to fetch repositories: {response.status_code} - {response.text}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(jfrog_url, repo_name, headers):
    url = f"{jfrog_url}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json().get("files", [])
    elif response.status_code == 404:
        print(f"⚠️ Skipping repository {repo_name}: Not Found (404)")
    else:
        print(f"❌ Failed to list artifacts for {repo_name}: {response.status_code} - {response.text}")
    
    return []

# Function to process repository artifacts
def process_repository(jfrog_url, repo_name, package_type, headers):
    artifact_list = list_artifacts(jfrog_url, repo_name, headers)
    repo_details = []

    if not artifact_list:
        return repo_details  # Skip empty repositories

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{jfrog_url}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Parse package name and version from artifact path
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 2:
            package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
            version = segments[-2]
        else:
            package_name = repo_name
            version = ""

        # Append only Docker repositories
        if package_type == "Docker":
            repo_details.append({
                "repo_name": repo_name,
                "package_type": package_type,
                "package_name": package_name,
                "version": version,
                "url": artifact_url,
                "created_date": created_date,
                "license": "",
                "secarch": "",
                "artifactory_instance": jfrog_url
            })
    
    return repo_details

# Function to save data to CSV
def save_to_csv(data, output_path):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    
    with open(output_path, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"✅ Data has been written to {output_path}")

# Main function
def main():
    parser = argparse.ArgumentParser(description="Generate JFrog Artifactory report")
    parser.add_argument("--org", required=True, help="Organization name (ev/ppa)")
    parser.add_argument("--jfrog-url", required=True, help="JFrog Artifactory URL")
    parser.add_argument("--output", required=True, help="Output directory for reports")
    args = parser.parse_args()

    # Set up authentication
    ARTIFACTORY_TOKEN = os.getenv("JFROG_API_KEY")
    if not ARTIFACTORY_TOKEN:
        print("❌ JFROG_API_KEY environment variable is not set.")
        exit(1)

    headers = {
        'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
        'Accept': 'application/json',
    }

    repositories = fetch_repositories(args.jfrog_url, headers)
    all_repo_details = []
    skipped_repos = []  # Track skipped repositories

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")
        print(f"🔍 Processing repository: {repo_name} (Type: {package_type})")

        if package_type == "Docker":
            try:
                repo_details = process_repository(args.jfrog_url, repo_name, package_type, headers)
                all_repo_details.extend(repo_details)
            except Exception as e:
                print(f"⚠️ Skipping repository {repo_name} due to error: {e}")
                skipped_repos.append(repo_name)

    # Save all details to CSV
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"{args.output}/EV_EOL_{current_date}_docker.csv"
    save_to_csv(all_repo_details, output_file)

    # Print skipped repositories
    if skipped_repos:
        print("⚠️ Skipped Repositories:")
        for repo in skipped_repos:
            print(f"  - {repo}")

if __name__ == '__main__':
    main()
```

---

### **What’s Updated?**
✅ **Catches errors** when processing repositories and logs them instead of failing the workflow.  
✅ **Logs skipped repositories** so they appear in GitHub Actions output.  
✅ **Uses environment variable `JFROG_API_KEY`** for authentication dynamically.  
✅ **Prints meaningful error messages** for debugging.  
✅ **Ensures the script runs for both `ev` and `ppa` organizations** without stopping due to a failure.  

This should now work **without failing the entire workflow** if a repository is inaccessible. 🚀  

Let me know if you need any modifications!
