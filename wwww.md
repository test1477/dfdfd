To correctly extract the **`package_name`** as the full repository path (e.g., `msartaz-train-docker-pr/ace-test/test-github-java`) and extract the **`version`** correctly (e.g., `1.0.18`), we need to refine the logic in the script. The current issue arises because the `parse_artifact_details` function is treating the entire artifact path as the package name or version, which is incorrect.

---

### Plan to Fix:
1. **Correctly Identify `package_name`**:
   - The `package_name` should correspond to the repository path up to the version folder (excluding the version itself and any additional markers or files).
   - Example: For a path like `ace-test/test-github-java/1.0.18/sha256.marker`, the `package_name` should be `msartaz-train-docker-pr/ace-test/test-github-java`.

2. **Correctly Extract `version`**:
   - The `version` should be the segment immediately preceding the artifact file or marker (e.g., `1.0.18` from `ace-test/test-github-java/1.0.18/sha256.marker`).

3. **Exclude Markers/Files from the Version Logic**:
   - Files like `sha256.marker` are not versions; the script must handle such cases.

---

### Updated Script with Refined Logic

Here’s the updated script:

```python
import requests
import csv
import warnings
from urllib.parse import urljoin

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

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

# Function to fetch metadata for a single artifact
def fetch_metadata(repo_name, artifact_path):
    url = f"{JFROG_URL}/api/storage/{repo_name}{artifact_path}"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch metadata for {repo_name}{artifact_path}: {response.status_code}")
        return {}

# Function to parse artifact details to extract package name and version
def parse_artifact_details(repo_name, artifact_path):
    """
    Extract package name and version from the artifact path.
    - Package name: Full repository path up to the version folder.
    - Version: Folder or segment immediately preceding the file/marker.
    """
    segments = artifact_path.strip("/").split("/")
    if len(segments) >= 2:
        # Package name: Everything except the last two segments
        package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
        # Version: Second-to-last segment
        version = segments[-2]
        return package_name, version
    return f"{repo_name}/N/A", "N/A"

# Function to process repository artifacts and extract metadata
def process_repository(repo_name):
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        metadata = fetch_metadata(repo_name, artifact_path)

        # Map fields from metadata
        package_name, version = parse_artifact_details(repo_name, artifact_path)
        repo_details.append({
            "repo_name": metadata.get("repo", repo_name),
            "package_type": "Docker",  # Customize this if necessary
            "package_name": package_name,
            "version": version,
            "url": metadata.get("downloadUri", "N/A"),
            "created_date": metadata.get("created", "N/A"),
            "artifactory_instance": "frigate.jfrog.io"
        })
    
    return repo_details

# Function to save data to CSV
def save_to_csv(data, filename='jfrog_artifact_metadata.csv'):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'artifactory_instance']
    
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

# Main function to iterate over repositories and fetch metadata
def main():
    repositories = ["msartaz-train-docker-pr"]  # Replace with your repository names

    all_repo_details = []
    for repo_name in repositories:
        print(f"Processing repository: {repo_name}")
        repo_details = process_repository(repo_name)
        all_repo_details.extend(repo_details)

    save_to_csv(all_repo_details)

if __name__ == '__main__':
    main()
```

---

### Explanation of Changes:

1. **Updated `parse_artifact_details`**:
   - Extracts the **`package_name`** as the full repository path up to the version folder.
     - For example:
       - `artifact_path = "ace-test/test-github-java/1.0.18/sha256.marker"`
       - `package_name = "msartaz-train-docker-pr/ace-test/test-github-java"`
   - Extracts the **`version`** as the segment immediately preceding the artifact file or marker.
     - Example:
       - `version = "1.0.18"`

2. **Adjusted `process_repository`**:
   - Iterates over all artifacts in the repository, fetching and mapping metadata correctly.
   - Uses the refined `parse_artifact_details` to set `package_name` and `version`.

3. **Handles Nested Repository Structures**:
   - Correctly identifies paths like `ace-test/test-github-java/1.0.18/sha256.marker` and separates package paths from versions.

---

### Example Input and Output:

#### Example Input (Repository Structure):
```plaintext
msartaz-train-docker-pr/
    ace-test/
        test-github-java/
            1.0.18/
                sha256.marker
            1.0.19/
                sha256.marker
```

#### Example Output (CSV):
| repo_name               | package_type | package_name                                | version  | url                                                                                           | created_date              | artifactory_instance |
|-------------------------|--------------|---------------------------------------------|----------|-----------------------------------------------------------------------------------------------|---------------------------|-----------------------|
| msartaz-train-docker-pr | Docker       | msartaz-train-docker-pr/ace-test/test-github-java | 1.0.18   | https://frigate.jfrog.io/artifactory/msartaz-train-docker-pr/ace-test/test-github-java/1.0.18/sha256.marker | 2024-05-01T17:45:42.3832 | frigate.jfrog.io     |
| msartaz-train-docker-pr | Docker       | msartaz-train-docker-pr/ace-test/test-github-java | 1.0.19   | https://frigate.jfrog.io/artifactory/msartaz-train-docker-pr/ace-test/test-github-java/1.0.19/sha256.marker | 2024-05-01T17:45:42.3832 | frigate.jfrog.io     |

---

### Testing and Verification:

1. **Run the Script**:
   - Replace `your_artifactory_api_token` with your actual API token.
   - Update the `repositories` list with the names of the repositories you want to process.

2. **Validate the Output**:
   - Check the `package_name` and `version` in the generated `jfrog_artifact_metadata.csv` file.
   - Ensure they match your expectations.

Let me know how it works and if further adjustments are needed!
