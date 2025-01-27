Here’s the full script with all the necessary updates to handle package name and version extraction, including a default handler for unsupported types to avoid `NotImplementedError`.

---

### Full Python Script

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

# Abstract base class for package type handlers
class PackageTypeHandler:
    """
    Base class for all package type handlers.
    Provides a default implementation for get_package_and_version.
    """
    def __init__(self, repo_name):
        self.repo_name = repo_name
        self.patterns = []

    def normalize_name(self, artifact_name):
        """
        Normalize artifact names by removing extensions like .tar.gz or .tgz.
        """
        match = re.match(r"(.*)\.(tar\.gz|tgz|json|zip|jar|nupkg)", artifact_name)
        return match.group(1) if match else artifact_name

    def get_package_and_version(self, artifact):
        """
        Default implementation: Extracts the package name and version from artifact path.
        Assumes the structure: <repo>/<package>/<version>/<file>.
        """
        artifact_path = artifact['path'].strip("/")
        segments = artifact_path.split("/")
        
        if len(segments) >= 3:
            # Generic logic: <repo>/<package>/<version>/<file>
            package_name = "/".join(segments[:-2])
            version = segments[-2]
            return package_name, version

        # If the structure doesn't match, return None
        logging.warning(f"Could not extract package and version for artifact: {artifact_path}")
        return None, None

# NPM-specific handler
class NpmTypeHandler(PackageTypeHandler):
    def __init__(self, repo_name):
        super().__init__(repo_name)
        self.patterns = [
            r"^(.*)-([\.\d]*)$",  # Matches <name>-<version>
            r"^(.*)-([\.\d]*)-.*$",  # Matches <name>-<version>-<suffix>
            r"^(.*)-(master|main|dev|alpha|beta|rc|canary|next|preview|latest|HEAD)$",  # Matches special cases
            r"^(.*)-([a-f0-9]*)$",  # Matches <name>-<hash>
        ]

    def get_package_and_version(self, artifact):
        artifact_name = self.normalize_name(artifact['name'])

        for pattern in self.patterns:
            match = re.match(pattern, artifact_name)
            if match:
                package_name = match.group(1)
                version = match.group(2) if len(match.groups()) > 1 else None
                return package_name, version

        # Default to fallback if no patterns matched
        return artifact_name, None

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

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    # Use NpmTypeHandler for npm repositories; fallback to PackageTypeHandler otherwise
    handler = NpmTypeHandler(repo_name) if package_type == "npm" else PackageTypeHandler(repo_name)
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Extract package name and version
        package_name, version = handler.get_package_and_version(artifact)
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

### Key Features:

1. **NpmTypeHandler**:
   - Custom regex patterns for npm artifacts like `package-name-1.0.0.json`.

2. **Fallback Handler**:
   - The base `PackageTypeHandler` provides default behavior for unsupported types.

3. **Debug Logging**:
   - Logs skipped artifacts for better troubleshooting.

---

### Usage:

1. Replace `your_artifactory_api_token` with your actual JFrog API token.
2. Run the script in Python:
   ```bash
   python jfrog_repo_parser.py
   ```
3. Check the generated CSV file in the same directory with a name like `EV_EOL_YYYY-MM-DD.csv`.

Let me know how it performs! 😊
