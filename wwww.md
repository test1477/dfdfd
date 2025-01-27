The issue arises because the current logic for extracting `package_name` and `version` is not tailored for the specific structure of npm artifacts in your repository. Specifically:

1. **NPM Package Structure**:
   - NPM URLs often include `.npm/<package-name>/<package-file>`.
   - The `package_name` should be extracted from the `<package-name>` segment.
   - The `version` should be extracted from the file name, typically formatted as `<package-name>-<version>.json`.

2. **Incorrect Parsing**:
   - The current script treats the segments leading up to `.npm` as the package name, which is incorrect.
   - It also mistakenly treats the `<package-name>` as the version.

---

### Fix for Parsing NPM Package Names and Versions

We need to:
1. Identify URLs containing `.npm/`.
2. Extract:
   - **`package_name`**: The segment following `.npm/`.
   - **`version`**: The version from the file name (e.g., `ppa-fi-qsg-documentation-client-1.0.1.json` → `1.0.1`).

---

### Updated Script for Correct NPM Parsing

Here’s the revised script:

```python
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime
import re  # For regex to parse versions

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

# Function to parse package name and version
def parse_artifact_details(repo_name, artifact_path, package_type):
    """
    Extracts package_name and version based on repository structure and type.
    """
    artifact_path = artifact_path.strip("/")
    if package_type == "npm" and ".npm/" in artifact_path:
        # Special handling for npm artifacts
        match = re.search(r"\.npm/([^/]+)/([^/]+)-([\d\.]+)\.json", artifact_path)
        if match:
            package_name = match.group(1)  # Extract package name
            version = match.group(3)  # Extract version
            return package_name, version
    else:
        # Generic parsing for other types
        segments = artifact_path.split("/")
        if len(segments) >= 2:
            package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
            version = segments[-2]
            return package_name, version
    return repo_name, ""  # Default if parsing fails

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

        # Parse package name and version
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
        
        # Process repository artifacts
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

### Key Updates:

1. **Special Parsing for NPM Artifacts**:
   - For paths containing `.npm/`, the script uses a regular expression to extract:
     - `package_name`: The segment immediately following `.npm/`.
     - `version`: Extracted from the file name (e.g., `ppa-fi-qsg-documentation-client-1.0.1.json` → `1.0.1`).

2. **Generic Parsing for Other Types**:
   - Artifacts not under `.npm/` use the generic parsing logic.

3. **Dynamic Package Type Handling**:
   - The `package_type` is dynamically passed into `parse_artifact_details` for type-specific parsing.

---

### Example Input and Output:

#### Input NPM Artifact:
```plaintext
.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json
```

#### Parsed Output:
| repo_name          | package_type | package_name                     | version  | url                                                                                   | created_date              | license | secarch | artifactory_instance |
|--------------------|--------------|-----------------------------------|----------|---------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| npm-ppa-virtual    | npm          | ppa-fi-qsg-documentation-client  | 1.0.1    | https://frigate.jfrog.io/artifactory/npm-ppa-virtual/.npm/ppa-fi-qsg-documentation-client/ppa-fi-qsg-documentation-client-1.0.1.json | 2024-01-15T14:20:50.1234 |         |         | frigate.jfrog.io     |

---

### Testing:
1. **Run the Script**:
   - Replace `your_artifactory_api_token` with your actual token.
   - Verify that the `package_name` and `version` fields are parsed correctly for NPM repositories.

2. **Provide Additional Examples**:
   - Share other repository structures if any further adjustments are needed. 

Let me know if this resolves the issue! 😊
