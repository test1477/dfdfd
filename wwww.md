To ensure the **package name** and **version** are extracted correctly for **npm artifacts**, such as in the given URL (`https://frigate.jfrog.io/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json`), we need to adjust the logic in the `parse_artifact_details` function.

---

### Updated Logic for NPM Artifacts

For **npm artifacts**:
1. **Package Name**: The segment before the version in the artifact's filename.
   - Example: `ppa-fi-qsg-documentation-client` (extracted from `ppa-fi-qsg-documentation-client-1.0.1.json`).
2. **Version**: The version embedded in the artifact filename.
   - Example: `1.0.1` (extracted from `ppa-fi-qsg-documentation-client-1.0.1.json`).

This requires detecting and parsing the artifact filename in the path (e.g., `ppa-fi-qsg-documentation-client-1.0.1.json`).

---

### Adjusted Script

Here’s the updated script with refined logic for handling npm artifacts:

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

# Function to extract package name and version from artifact path
def parse_artifact_details(repo_name, artifact_path, package_type):
    """
    Extracts package name and version based on the artifact path and package type.
    - For npm artifacts, extracts from the filename.
    - For other types, uses the default logic.
    """
    artifact_filename = artifact_path.strip("/").split("/")[-1]

    if package_type == "npm" and artifact_filename:
        # Extract npm package name and version using regex
        match = re.match(r"(.+)-(\d+\.\d+\.\d+)\.json", artifact_filename)
        if match:
            package_name, version = match.groups()
            return package_name, version

    # Default logic for non-npm repositories
    segments = artifact_path.strip("/").split("/")
    if len(segments) >= 2:
        package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
        version = segments[-2]
        return package_name, version

    return repo_name, ""

# Function to process repository artifacts
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

        # Parse package name and version based on package type
        package_name, version = parse_artifact_details(repo_name, artifact_path, package_type)

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
        
        # Process repository artifacts in batches
        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = f"EV_EOL_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### Key Changes:

1. **Updated `parse_artifact_details`**:
   - For npm artifacts, uses a regular expression (`re.match`) to extract the package name and version from the artifact filename.
   - Example:
     - Artifact: `ppa-fi-qsg-documentation-client-1.0.1.json`
     - Extracted `package_name`: `ppa-fi-qsg-documentation-client`
     - Extracted `version`: `1.0.1`

2. **Default Logic for Non-NPM Repositories**:
   - Retains the existing logic for other repository types (e.g., Docker, Helm).

3. **Regex Pattern for NPM Artifacts**:
   - Matches filenames in the format `<package_name>-<version>.json`.

---

### Example Output (CSV):

For an npm repository with the following artifact:

```plaintext
https://frigate.jfrog.io/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json
```

The output would look like this:

| repo_name               | package_type | package_name                      | version  | url                                                                                         | created_date              | license | secarch | artifactory_instance |
|-------------------------|--------------|-----------------------------------|----------|---------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| npm-ppa-virtual         | npm          | ppa-fi-qsg-documentation-client  | 1.0.1    | https://frigate.jfrog.io/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json | 2025-01-24T14:20:50.1234 |         |         | frigate.jfrog.io     |

---

### Next Steps:

1. **Test the Script**:
   - Run the script with your Artifactory instance.
   - Verify the extracted `package_name` and `version` fields for npm artifacts.

2. **Add Additional Parsing (if needed)**:
   - Let me know if other repository types (e.g., Helm, Docker) require custom parsing logic.

Let me know how it works for you! 😊
