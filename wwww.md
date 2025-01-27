Let’s diagnose and fix the issue for the non-Docker package types. From your input, it seems like:
1. **Npm artifacts** are not extracting the `package_name` and `version` properly.
2. Other package types might also need specific handlers or better logic.

I’ll simplify and refine the code while ensuring compatibility with npm and other common types like Maven, Helm, and NuGet. Here's the corrected script:

---

### Key Adjustments:
1. **Explicit Handling for npm**:
   - `package_name` and `version` are extracted from the filename for `.json` files.
   - Regex fixes to capture relevant data.
   
2. **Fallback for Other Types**:
   - For Maven, Helm, etc., the logic extracts data from the path structure.

3. **Debugging**:
   - Added logging to help identify which files are not matching expected patterns.

---

### Updated Script

```python
import requests
import csv
import warnings
import re
import logging
from urllib.parse import urljoin
from datetime import datetime

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# Logging setup for debugging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# JFrog Artifactory instance details
JFROG_URL = 'https://frigate.jfrog.io/artifactory'  # Base URL for JFrog
ARTIFACTORY_TOKEN = 'your_artifactory_api_token'  # Replace with your API token

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

# Function to fetch all repositories
def fetch_repositories():
    url = f"{JFROG_URL}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json()
    else:
        logging.error(f"Failed to fetch repositories: {response.status_code}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        logging.error(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

# Function to extract package name and version
def extract_package_details(repo_name, artifact_path, package_type):
    """
    Extracts package name and version based on the repository type and artifact path.
    """
    artifact_filename = artifact_path.strip("/").split("/")[-1]

    if package_type == "npm":
        # Extract package name and version from npm artifact filenames
        match = re.match(r"^(.*)-(\d+\.\d+\.\d+)\.json$", artifact_filename)
        if match:
            package_name, version = match.groups()
            return package_name, version

    elif package_type == "maven":
        # Maven package structure: group/artifact/version/artifact-version.ext
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 3:
            package_name = "/".join(segments[:-2])
            version = segments[-2]
            return package_name, version

    elif package_type == "helm":
        # Helm charts: chartname-version.tgz
        match = re.match(r"^(.*)-(\d+\.\d+\.\d+)\.tgz$", artifact_filename)
        if match:
            package_name, version = match.groups()
            return package_name, version

    elif package_type == "nuget":
        # NuGet packages: name/version/name-version.nupkg
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 3:
            package_name = segments[-3]
            version = segments[-2]
            return package_name, version

    # Default fallback for other types
    segments = artifact_path.strip("/").split("/")
    if len(segments) >= 2:
        package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
        version = segments[-2]
        return package_name, version

    # If no match is found, return None
    logging.warning(f"Could not parse package details for: {artifact_path}")
    return None, None

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Extract package name and version
        package_name, version = extract_package_details(repo_name, artifact_path, package_type)

        if package_name and version:
            repo_details.append({
                "repo_name": repo_name,
                "package_type": package_type,
                "package_name": package_name,
                "version": version,
                "url": artifact_url,
                "created_date": created_date,
                "license": "",  # Leave blank if not available
                "secarch": "",  # Leave blank if not available
                "artifactory_instance": "frigate.jfrog.io",
            })

    return repo_details

# Function to save data to CSV
def save_to_csv(data, filename):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    print(f"Data written to {filename}")

# Main function to process repositories
def main():
    repositories = fetch_repositories()
    all_repo_details = []

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")
        print(f"Processing repository: {repo_name} (Type: {package_type})")

        # Process repository artifacts
        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    # Save to CSV
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"EV_EOL_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == "__main__":
    main()
```

---

### Key Improvements:
1. **Regex Refinement**:
   - Npm: Extracts `package_name` and `version` from filenames like `package-1.2.3.json`.
   - Helm: Matches `chart-1.2.3.tgz`.
   - Maven: Extracts details from folder structure.

2. **Default Fallback**:
   - Handles cases where `package_type` is unknown or not explicitly supported.

3. **Debug Logging**:
   - Warns when the script cannot parse `package_name` or `version`.

---

### Example Outputs:

#### Input:
- **npm**: `/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json`
- **Maven**: `/group/artifact/1.2.3/artifact-1.2.3.jar`
- **Helm**: `/charts/chartname-2.0.0.tgz`

#### Output (CSV):
| repo_name       | package_type | package_name                      | version  | url                                                                                          | created_date              | license | secarch | artifactory_instance |
|-----------------|--------------|-----------------------------------|----------|----------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| npm-ppa-virtual | npm          | ppa-fi-qsg-documentation-client  | 1.0.1    | https://frigate.jfrog.io/npm-ppa-virtual/ppa-fi-qsg-documentation-client-1.0.1.json          | 2025-01-24T14:20:50.1234 |         |         | frigate.jfrog.io     |
| maven-central   | maven        | group/artifact                   | 1.2.3    | https://frigate.jfrog.io/maven-central/group/artifact/1.2.3/artifact-1.2.3.jar              | 2025-01-24T15:00:00.0000 |         |         | frigate.jfrog.io     |
| helm-repo       | helm         | chartname                        | 2.0.0    | https://frigate.jfrog.io/helm-repo/charts/chartname-2.0.0.tgz                                | 2025-01-24T16:30:00.0000 |         |         | frigate.jfrog.io     |

---

### Next Steps:
1. **Test the Script**:
   - Ensure npm, Maven, Helm, and other package types extract `package_name` and `version` correctly.
2. **Log Issues**:
   - Use logging to identify any artifacts that do not match expected patterns.

Let me know if it works or needs further adjustments! 😊
