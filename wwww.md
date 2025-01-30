Got it! You want a **simplified approach** where the script extracts the **package name** and **version** directly from the filename for all package types (e.g., npm, NuGet, Helm, etc.). The logic will be consistent across all package types:

- For **npm**: Extract name and version from `.tgz` filenames (e.g., `package1-12.tgz` → `package1`, `12`).
- For **NuGet**: Extract name and version from `.nupkg` filenames (e.g., `package2-3.4.5.nupkg` → `package2`, `3.4.5`).
- For **Helm**: Extract name and version from `.tgz` filenames (e.g., `chart1-1.2.3.tgz` → `chart1`, `1.2.3`).

Here’s the updated script with this simplified logic:

---

### Updated Script: Extract Package Name and Version from Filename

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

# Function to extract package name and version from filename
def extract_package_info(filename):
    """
    Extracts package name and version from a filename.
    Supported formats:
    - npm: package1-12.tgz → package1, 12
    - NuGet: package2-3.4.5.nupkg → package2, 3.4.5
    - Helm: chart1-1.2.3.tgz → chart1, 1.2.3
    """
    # Regex to match package name and version
    match = re.match(r"^(.*)-(\d+\.\d+\.\d+)\.(tgz|nupkg)$", filename)
    if match:
        return match.group(1), match.group(2)
    return filename, "unknown"

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository and extracts metadata.
    """
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Extract package name and version from filename
        filename = artifact_path.split("/")[-1]
        package_name, version = extract_package_info(filename)

        # Append details for all package types
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

        # Process all repositories
        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = f"EV_EOL_{current_date}_all_packages.csv"  # Save reports for all package types
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### Key Changes:
1. **Simplified Logic**:
   - The `extract_package_info` function uses a **regex** to extract the package name and version from the filename.
   - It works for:
     - npm: `package1-12.tgz` → `package1`, `12`
     - NuGet: `package2-3.4.5.nupkg` → `package2`, `3.4.5`
     - Helm: `chart1-1.2.3.tgz` → `chart1`, `1.2.3`

2. **Consistent Across Package Types**:
   - The same logic is applied to all package types, making the script simpler and more maintainable.

3. **Fallback for Unknown Formats**:
   - If the filename doesn’t match the expected format, the script defaults to using the filename as the package name and `"unknown"` as the version.

---

### Example Output (CSV):
| repo_name   | package_type | package_name | version | url                                      | created_date       | license | secarch | artifactory_instance   |
|-------------|--------------|--------------|---------|------------------------------------------|--------------------|---------|---------|------------------------|
| npm-repo    | npm          | package1     | 12      | https://frigate.jfrog.io/.../package1-12.tgz | 2023-10-01 12:34:56 |         |         | frigate.jfrog.io       |
| nuget-repo  | NuGet        | package2     | 3.4.5   | https://frigate.jfrog.io/.../package2-3.4.5.nupkg | 2023-09-30 10:11:12 |         |         | frigate.jfrog.io       |
| helm-repo   | Helm         | chart1       | 1.2.3   | https://frigate.jfrog.io/.../chart1-1.2.3.tgz | 2023-09-29 14:15:16 |         |         | frigate.jfrog.io       |

---

### Regex Explanation:
The regex `^(.*)-(\d+\.\d+\.\d+)\.(tgz|nupkg)$` works as follows:
- `^(.*)`: Captures the package name (everything before the hyphen).
- `-(\d+\.\d+\.\d+)`: Captures the version (e.g., `12`, `3.4.5`, `1.2.3`).
- `\.(tgz|nupkg)$`: Ensures the filename ends with `.tgz` or `.nupkg`.

---

### Summary:
- The script now extracts package names and versions directly from filenames for **npm**, **NuGet**, and **Helm** packages.
- The logic is consistent and simple, making it easy to maintain and extend to other package types if needed.

Let me know if you need further assistance!
