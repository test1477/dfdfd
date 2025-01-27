Certainly! If you'd like to restrict the script to process only repositories of type **Docker**, we can add a filter to the repository processing logic. This will ensure that only Docker repositories are included.

---

### Updated Script: Process Only Docker Repositories

Here's the script adjusted to handle only Docker repositories:

```python
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime
import logging

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

# Function to process Docker repositories
def process_docker_repository(repo_name):
    """
    Processes all artifacts in a Docker repository and extracts metadata.
    """
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        # Extract Docker-specific package name and version
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 2:
            package_name = f"{repo_name}/{segments[0]}"
            version = segments[1] if len(segments) > 1 else "latest"
        else:
            package_name = repo_name
            version = "latest"

        # Append details
        repo_details.append({
            "repo_name": repo_name,
            "package_type": "Docker",
            "package_name": package_name,
            "version": version,
            "url": artifact_url,
            "created_date": created_date,
            "license": "",  # Leave blank for now
            "secarch": "",  # Leave blank for now
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

# Main function to process repositories
def main():
    """
    Main function to process all Docker repositories and extract metadata.
    """
    repositories = fetch_repositories()
    all_repo_details = []

    # Filter and process only Docker repositories
    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")
        if package_type == "docker":  # Only process Docker repositories
            print(f"Processing Docker repository: {repo_name}")
            repo_details = process_docker_repository(repo_name)
            all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"EV_DOCKER_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### Key Changes:

1. **Filter for Docker Repositories**:
   - In the `main` function, the script now checks if the `packageType` is `"docker"`. Only repositories of this type are processed.

2. **Extract Package Name and Version**:
   - For Docker repositories, the **package name** is derived from the first segment of the artifact path.
   - The **version** is derived from the second segment (e.g., `repo/image/version`).

3. **Dynamic File Name**:
   - The output file is named `EV_DOCKER_<current_date>.csv`, where `<current_date>` is the current date.

---

### Example Input:

**Repository**: `docker-example`
**Artifacts**:
```plaintext
/redis/latest/manifest.json
/redis/6.0/manifest.json
```

---

### Example Output (CSV):

| repo_name       | package_type | package_name      | version | url                                                               | created_date              | license | secarch | artifactory_instance |
|-----------------|--------------|-------------------|---------|-------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| docker-example  | Docker       | docker-example/redis | latest  | https://frigate.jfrog.io/artifactory/docker-example/redis/latest/manifest.json | 2025-01-24T12:34:56.789Z |         |         | frigate.jfrog.io     |
| docker-example  | Docker       | docker-example/redis | 6.0     | https://frigate.jfrog.io/artifactory/docker-example/redis/6.0/manifest.json    | 2025-01-24T12:34:56.789Z |         |         | frigate.jfrog.io     |

---

### Next Steps:

1. **Test the Script**:
   - Replace `your_artifactory_api_token` with a valid API token.
   - Run the script to confirm that it processes only Docker repositories.

2. **Enhance Parsing**:
   - If Docker artifact paths are more complex, adjust the `process_docker_repository` function to handle additional patterns.

Let me know if you need further adjustments! 😊
