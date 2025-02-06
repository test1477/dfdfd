Got it! The **`repo_name` (e.g., `docker-images`) was missing from the `package_name` extraction** but should be present in the `url`. I’ve fixed that.

---

### **Final Fixes:**
✅ **Ensure `repo_name` is correctly included in the URL**  
✅ **Ensure `package_name` includes everything after `repo_name`, but before the version**  
✅ **Avoid `.jfrog` cache folders**  

---

### **🚀 Full Corrected Script**
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
JFROG_URL = args.jfrog_url.rstrip("/")  # Ensure no trailing slash
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
    Processes all artifacts in a repository while avoiding `.jfrog` cache folders.
    """
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "").lstrip("/")  # Remove leading slash if present

            # Skip .jfrog cache folder
            if ".jfrog" in artifact_path:
                continue

            # Extract path segments
            path_segments = artifact_path.split("/")

            # Extract version (last directory before filename)
            if len(path_segments) >= 2:
                version = path_segments[-2]
            else:
                version = ""

            # Extract package name (everything after repo_name but before the version)
            package_name = "/".join(path_segments[:-2]) if len(path_segments) > 2 else ""

            # Ensure package name is correctly formatted
            if package_name.startswith(repo_name + "/"):
                package_name = package_name[len(repo_name) + 1:]

            # Construct the full artifact URL (ensuring repo_name is included)
            artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"

            # Store only relevant Docker repository data
            if package_type == "Docker":
                repo_details.append({
                    "repo_name": repo_name,
                    "package_type": package_type,
                    "package_name": package_name,
                    "version": version,
                    "url": artifact_url,
                    "created_date": artifact.get("lastModified", ""),
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

### **🔧 Fixes & Enhancements**
✅ **Ensures `repo_name` is included in the URL**  
- **Fix:** `artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"`  

✅ **Ensures `package_name` correctly excludes `repo_name` but includes the full path before the version**  
- Example Fix:  
  ```python
  package_name = "/".join(path_segments[:-2]) if len(path_segments) > 2 else ""
  if package_name.startswith(repo_name + "/"):
      package_name = package_name[len(repo_name) + 1:]
  ```
- **Before (Incorrect Package Name):**  
  ```
  eaton-vance-corp/angular-apm-dd/frontend/angular-apm-dd/v1.0.0/manifest
  ```
- **Now (Correct Package Name):**  
  ```
  eaton-vance-corp/angular-apm-dd/frontend/angular-apm-dd
  ```

✅ **Skips `.jfrog` cache folders**  

✅ **Outputs a properly formatted CSV**  

---

### **📌 Final Example Output (CSV)**
| repo_name      | package_type | package_name                                  | version  | url                                                                                   | created_date | license | secarch | artifactory_instance |
|---------------|-------------|-----------------------------------------------|---------|--------------------------------------------------------------------------------------|--------------|---------|---------|----------------------|
| docker-images | Docker      | eaton-vance-corp/angular-apm-dd/frontend/angular-apm-dd | v1.0.0  | https://frigate.jfrog.io/artifactory/docker-images/eaton-vance-corp/.../manifest.json | 2024-02-05   |         |         | frigate.jfrog.io     |

---

### **🚀 Summary**
✅ Fixes the `repo_name` issue in the **URL**  
✅ Fixes **`package_name` extraction** (removes `repo_name` but keeps everything before the version)  
✅ Properly skips `.jfrog` cache folders  

This should now be fully correct! Let me know if you need any more refinements. 😊
