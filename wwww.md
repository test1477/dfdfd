It seems the logic for parsing the `package_name` and `version` from the artifact paths in npm repositories isn't working correctly. The issue is likely due to the structure of the artifact URL and the assumptions made in the parsing logic.

Let's take a step-by-step approach to debug and refine the script:

---

### Observations:
1. **`version` Issue**:
   - Instead of extracting the actual version (e.g., `1.0.1`), it's extracting part of the artifact name (`ppa-fi-qsg-documentation-client`).

2. **`package_name` Issue**:
   - The script incorrectly sets the package name to `.npm` (a part of the path), rather than the actual package name (`ppa-fi-qsg-documentation-client`).

---

### Revised Approach:

We need to:
1. **Inspect the Full Path**:
   - Use the entire path to locate both the package name and version.
2. **Add Better Logic for `npm` Repositories**:
   - Extract `package_name` and `version` correctly based on the artifact path.
3. **Debugging**:
   - Print artifact paths during processing to confirm the structure.

---

### Updated Script

Here’s the revised script with improved handling for npm repositories:

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

# Function to extract package name and version for npm repositories
def extract_npm_package_details(repo_name, artifact_path):
    """
    Extracts package name and version for npm repositories.
    Example: /ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json
    """
    artifact_filename = artifact_path.strip("/").split("/")[-1]
    match = re.match(r"^(.*)-(\d+\.\d+\.\d+)\.json$", artifact_filename)
    if match:
        package_name, version = match.groups()
        return package_name, version

    # If no match, log the issue for debugging
    logging.warning(f"Could not parse npm package details: {artifact_path}")
    return None, None

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Extract package name and version based on type
        if package_type == "npm":
            package_name, version = extract_npm_package_details(repo_name, artifact_path)
        else:
            # Default logic for other package types
            segments = artifact_path.strip("/").split("/")
            if len(segments) >= 2:
                package_name = "/".join(segments[:-2])
                version = segments[-2]
            else:
                package_name, version = None, None

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
        else:
            logging.warning(f"Skipping artifact: {artifact_path} (Could not extract package details)")

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

        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"EV_EOL_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == "__main__":
    main()
```

---

### Changes Made:
1. **Improved NPM Parsing**:
   - The `extract_npm_package_details` function explicitly matches the filename for npm packages (e.g., `ppa-fi-qsg-documentation-client-1.0.1.json`) and extracts:
     - **`package_name`**: `ppa-fi-qsg-documentation-client`
     - **`version`**: `1.0.1`

2. **Default Fallback**:
   - Other package types use folder structure to extract the `package_name` and `version`.

3. **Debugging**:
   - Warnings are logged for artifacts that don’t match the expected structure.

---

### Example Outputs:

#### Input (NPM Artifact):
Artifact Path: `/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json`

#### Output (CSV):
| repo_name       | package_type | package_name                      | version  | url                                                                                          | created_date              | license | secarch | artifactory_instance |
|-----------------|--------------|-----------------------------------|----------|----------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| npm-ppa-virtual | npm          | ppa-fi-qsg-documentation-client  | 1.0.1    | https://frigate.jfrog.io/npm-ppa-virtual/ppa-fi-qsg-documentation-client-1.0.1.json          | 2025-01-24T14:20:50.1234 |         |         | frigate.jfrog.io     |

---

### Debugging Tips:
1. **Print Artifact Paths**:
   - Add a `print(artifact_path)` line in the `process_repository` function to inspect the structure.
2. **Check Logs**:
   - Look for warnings about artifacts that could not be parsed.

---

### Next Steps:
1. **Run the Script**:
   - Use your API token and Artifactory instance.
2. **Validate Output**:
   - Check if the `package_name` and `version` are now correct for npm artifacts.

Let me know how this works! 😊
