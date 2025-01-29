To fetch **all package types** (e.g., npm, NuGet, Yum, Generic, Helm, Terraform, etc.) from JFrog Artifactory and correctly extract their metadata (including version and package type), we need to modify the script to handle different package types and their specific metadata structures.

Below is the updated Python script that fetches **all package types**, extracts their metadata, and saves the report in CSV format.

---

### Updated Python Script: `report_script.py`

```python
import requests
import csv
from datetime import datetime

# Configuration
ARTIFACTORY_URL = 'https://your-artifactory-instance/artifactory'  # Replace with your Artifactory URL
API_KEY = 'your-api-key'  # Replace with your API key or use username/password
CSV_FILE = 'all_packages_report.csv'

# Headers for authentication
headers = {
    'X-JFrog-Art-Api': API_KEY
}

def fetch_repositories():
    """Fetch all repositories from Artifactory."""
    repos_url = f'{ARTIFACTORY_URL}/api/repositories'
    response = requests.get(repos_url, headers=headers)
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

def fetch_packages(repo_key):
    """Fetch packages from a specific repository."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}'
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json().get('children', [])
    else:
        print(f"Failed to fetch packages from {repo_key}: {response.status_code}")
        return []

def fetch_package_metadata(repo_key, package_path):
    """Fetch metadata for a specific package."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}/{package_path}'
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch metadata for {package_path}: {response.status_code}")
        return None

def extract_package_info(repo_key, package_path, repo_type):
    """Extract package information based on repository type."""
    metadata = fetch_package_metadata(repo_key, package_path)
    if not metadata:
        return None

    package_name = metadata.get('name', 'N/A')
    version = metadata.get('version', 'N/A')
    url = metadata.get('downloadUri', 'N/A')
    created_date = metadata.get('created', 'N/A')

    # Convert timestamp to readable date
    if created_date != 'N/A':
        created_date = datetime.fromtimestamp(created_date / 1000).strftime('%Y-%m-%d %H:%M:%S')

    return [repo_key, repo_type, package_name, version, url, created_date]

def save_to_csv(data, filename):
    """Save the report data to a CSV file."""
    with open(filename, mode='w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(['Repo Name', 'Package Type', 'Package Name', 'Version', 'URL', 'Created Date'])
        for row in data:
            writer.writerow(row)

def main():
    # Fetch all repositories
    repositories = fetch_repositories()
    if not repositories:
        return

    report_data = []

    for repo in repositories:
        repo_key = repo['key']
        repo_type = repo['type']
        print(f"Processing repository: {repo_key} (Type: {repo_type})")

        packages = fetch_packages(repo_key)
        for pkg in packages:
            package_path = pkg['uri'][1:]  # Remove leading slash
            package_info = extract_package_info(repo_key, package_path, repo_type)

            if package_info:
                report_data.append(package_info)

    # Save the report to CSV
    save_to_csv(report_data, CSV_FILE)
    print(f"Report saved to {CSV_FILE}")

if __name__ == '__main__':
    main()
```

---

### Key Changes and Features:
1. **Supports All Package Types**:
   - The script now processes all repository types (e.g., npm, NuGet, Yum, Generic, Helm, Terraform, etc.).
   - The `repo_type` is extracted from the repository metadata and included in the report.

2. **Dynamic Metadata Extraction**:
   - The `extract_package_info` function dynamically extracts metadata for any package type.
   - It retrieves the `name`, `version`, `downloadUri`, and `created` fields from the package metadata.

3. **CSV Report**:
   - The report includes the following columns:
     - **Repo Name**: The name of the repository.
     - **Package Type**: The type of package (e.g., npm, NuGet, Docker, etc.).
     - **Package Name**: The name of the package.
     - **Version**: The version of the package.
     - **URL**: The URL to download the package.
     - **Created Date**: The date when the package was created.

4. **Handles All Repositories**:
   - The script fetches all repositories and processes each one, regardless of the package type.

---

### How to Use:
1. Replace `ARTIFACTORY_URL` with your JFrog Artifactory instance URL.
2. Replace `API_KEY` with your Artifactory API key or modify the script to use username/password for authentication.
3. Run the script:
   ```bash
   python report_script.py
   ```

---

### Example Output (CSV):
| Repo Name   | Package Type | Package Name     | Version | URL                                      | Created Date       |
|-------------|--------------|------------------|---------|------------------------------------------|--------------------|
| npm-repo    | npm          | my-package       | 1.2.3   | https://artifactory/.../my-package-1.2.3 | 2023-10-01 12:34:56 |
| docker-repo | docker       | my-image         | latest  | https://artifactory/.../my-image-latest  | 2023-09-30 10:11:12 |
| helm-repo   | helm         | my-chart         | 0.1.0   | https://artifactory/.../my-chart-0.1.0   | 2023-09-29 14:15:16 |

---

### Notes:
- The script assumes that the Artifactory API returns consistent metadata fields (`name`, `version`, `downloadUri`, `created`) for all package types. If some package types have different metadata structures, you may need to add custom logic to handle them.
- Ensure your API key has sufficient permissions to access all repositories and packages.
- If you encounter any issues with specific package types, let me know, and I can help refine the script further.

Let me know if you need additional assistance!
