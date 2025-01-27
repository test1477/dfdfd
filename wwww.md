To address this issue and properly handle npm package artifacts (like `ppa-fi-qsg-documentation-client-1.0.1.json`), we can implement a **package type handler** similar to the logic you provided. This involves creating a specific handler class for npm repositories that applies custom parsing logic to extract the **package name** and **version**.

Here's an implementation that incorporates your ideas while ensuring flexibility for other package types.

---

### Updated Script with a Package Type Handler

```python
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime
import re

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# JFrog Artifactory instance details
JFROG_URL = 'https://frigate.jfrog.io/artifactory'  # Base URL for JFrog
ARTIFACTORY_TOKEN = 'your_artifactory_api_token'  # Replace with your API token

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

# Abstract class for package type handlers
class PackageTypeHandler:
    def __init__(self, repo_name, artifact_path):
        self.repo_name = repo_name
        self.artifact_path = artifact_path

    def get_package_and_version(self):
        """
        Abstract method to be implemented by specific package type handlers.
        """
        raise NotImplementedError("This method should be overridden in the subclass.")

# Handler for npm repositories
class NpmTypeHandler(PackageTypeHandler):
    def get_package_and_version(self):
        """
        Extract package name and version for npm artifacts.
        Example: ppa-fi-qsg-documentation-client-1.0.1.json
        """
        artifact_filename = self.artifact_path.strip("/").split("/")[-1]
        match = re.match(r"^(.*)-(\d+\.\d+\.\d+)\.json$", artifact_filename)
        if match:
            package_name, version = match.groups()
            return package_name, version
        return "", ""  # Return empty strings if no match is found

# Handler for other repository types (default)
class DefaultTypeHandler(PackageTypeHandler):
    def get_package_and_version(self):
        """
        Default logic for extracting package name and version.
        Uses the directory structure of the artifact path.
        """
        segments = self.artifact_path.strip("/").split("/")
        if len(segments) >= 2:
            package_name = f"{self.repo_name}/{'/'.join(segments[:-2])}"
            version = segments[-2]
            return package_name, version
        return self.repo_name, ""

# Factory function to get the appropriate handler
def get_handler(repo_name, artifact_path, package_type):
    if package_type == "npm":
        return NpmTypeHandler(repo_name, artifact_path)
    return DefaultTypeHandler(repo_name, artifact_path)

# Function to fetch all repositories
def fetch_repositories():
    url = f"{JFROG_URL}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Use the appropriate handler to extract package name and version
        handler = get_handler(repo_name, artifact_path, package_type)
        package_name, version = handler.get_package_and_version()

        # Append details
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
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    print(f"Data has been written to {filename}")

# Main function to process all repositories
def main():
    repositories = fetch_repositories()
    all_repo_details = []

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")  # Dynamically fetch the package type
        print(f"Processing repository: {repo_name} (Type: {package_type})")
        
        # Process repository artifacts
        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"EV_EOL_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### Key Changes:

1. **Handler Classes**:
   - `NpmTypeHandler`:
     - Specifically designed to handle npm repositories.
     - Extracts `package_name` and `version` using a regex (`^(.*)-(\d+\.\d+\.\d+)\.json$`).
   - `DefaultTypeHandler`:
     - Handles all other package types using the default folder structure logic.

2. **Factory Function**:
   - `get_handler(repo_name, artifact_path, package_type)` dynamically selects the appropriate handler based on the `package_type`.

3. **Robust Parsing**:
   - Handles complex npm paths and ensures correct extraction of `package_name` and `version`.

---

### Example Output:

#### Input:
- **Artifact Path**: `https://frigate.jfrog.io/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json`

#### Output:
| repo_name       | package_type | package_name                      | version  | url                                                                                          | created_date              | license | secarch | artifactory_instance |
|-----------------|--------------|-----------------------------------|----------|----------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| npm-ppa-virtual | npm          | ppa-fi-qsg-documentation-client  | 1.0.1    | https://frigate.jfrog.io/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json | 2025-01-24T14:20:50.1234 |         |         | frigate.jfrog.io     |

---

### Benefits:
- Handles multiple package types with clean, extendable logic.
- Ensures accurate parsing for npm artifacts.

Let me know how it works! 😊
