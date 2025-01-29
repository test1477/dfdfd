The issue you're facing is that the script is not correctly extracting the **package type** and **version** for all package types (e.g., npm, NuGet, Yum, Generic, Helm, Terraform, etc.). This is because the script is currently hardcoded to handle **Docker** packages specifically, and the logic for extracting package names and versions may not work for other package types.

To fix this, we need to:
1. **Dynamically determine the package type** for each repository.
2. **Generalize the logic** for extracting package names and versions to work with all package types.
3. **Handle different package structures** (e.g., npm, NuGet, Docker, etc.) appropriately.

Here’s the updated script that works for **all package types**:

---

### Updated Script for All Package Types

```python
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime

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

# Function to extract package name and version based on package type
def extract_package_info(repo_name, package_type, artifact_path):
    """
    Extracts package name and version based on the package type and artifact path.
    """
    segments = artifact_path.strip("/").split("/")
    package_name = repo_name
    version = ""

    # Handle different package types
    if package_type == "Docker":
        # Docker: /<repo>/<image>/<tag>/manifest.json
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[-3]}"  # Image name
            version = segments[-2]  # Tag
    elif package_type == "npm":
        # npm: /<repo>/<package>/<version>/package.tgz
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[-2]}"  # Package name
            version = segments[-2]  # Version
    elif package_type == "NuGet":
        # NuGet: /<repo>/<package>/<version>/<package>.<version>.nupkg
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[-2]}"  # Package name
            version = segments[-2]  # Version
    elif package_type == "Generic":
        # Generic: /<repo>/<path>/<file>
        if len(segments) >= 1:
            package_name = f"{repo_name}/{'/'.join(segments[:-1])}"  # Path
            version = segments[-1]  # File name
    elif package_type == "Helm":
        # Helm: /<repo>/<chart>/<version>/<chart>-<version>.tgz
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[-2]}"  # Chart name
            version = segments[-2]  # Version
    elif package_type == "Terraform":
        # Terraform: /<repo>/<module>/<version>/<module>-<version>.zip
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[-2]}"  # Module name
            version = segments[-2]  # Version
    else:
        # Default behavior for unknown package types
        package_name = repo_name
        version = segments[-1] if segments else ""

    return package_name, version

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

        # Extract package name and version
        package_name, version = extract_package_info(repo_name, package_type, artifact_path)

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

        # Process all repositories (not just Docker)
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
1. **Dynamic Package Type Handling**:
   - The `extract_package_info` function now handles different package types (e.g., Docker, npm, NuGet, Generic, Helm, Terraform) and extracts the package name and version based on the repository type.

2. **Generalized Logic**:
   - The script processes **all package types**, not just Docker.
   - The logic for extracting package names and versions is tailored to the structure of each package type.

3. **CSV Output**:
   - The CSV file now includes metadata for **all package types**.

---

### Example Output (CSV):
| repo_name   | package_type | package_name     | version | url                                      | created_date       | license | secarch | artifactory_instance   |
|-------------|--------------|------------------|---------|------------------------------------------|--------------------|---------|---------|------------------------|
| npm-repo    | npm          | my-package       | 1.2.3   | https://frigate.jfrog.io/.../my-package  | 2023-10-01 12:34:56 |         |         | frigate.jfrog.io       |
| docker-repo | Docker       | my-image         | latest  | https://frigate.jfrog.io/.../my-image    | 2023-09-30 10:11:12 |         |         | frigate.jfrog.io       |
| helm-repo   | Helm         | my-chart         | 0.1.0   | https://frigate.jfrog.io/.../my-chart    | 2023-09-29 14:15:16 |         |         | frigate.jfrog.io       |

---

### Notes:
- The script assumes that the package structure follows standard conventions for each package type. If your Artifactory instance uses a custom structure, you may need to adjust the `extract_package_info` function.
- If you encounter any issues with specific package types, let me know, and I can help refine the logic further.

Let me know if you need additional assistance!
