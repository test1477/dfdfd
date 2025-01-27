If you only need to fetch reports for **Docker repositories**, we can filter the repositories based on their `packageType` and process only those with the `Docker` type. This will reduce unnecessary processing for other package types like `npm`, `helm`, etc.

Here’s the updated script that processes only Docker repositories:

---

### Updated Script for Only Docker Repositories

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

# Function to process repository artifacts without extra API calls
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

        # Parse package name and version from artifact path
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 2:
            package_name = f"{repo_name}/{'/'.join(segments[:-2])}"  # Up to the version folder
            version = segments[-2]  # Second-to-last segment
        else:
            package_name = repo_name
            version = ""

        # Append details only for Docker repositories
        if package_type == "Docker":
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

        # Process only Docker repositories
        if package_type == "Docker":
            repo_details = process_repository(repo_name, package_type)
            all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = f"EV_EOL_{current_date}_docker.csv"  # Save only Docker reports
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

---

### Key Updates:
1. **Filter for Docker Repositories**:
   - The script now processes **only repositories with `packageType == "Docker"`**.
   - This ensures that only Docker-related artifacts are processed and saved in the CSV file.

2. **Dynamic File Naming for Docker Reports**:
   - The output file is saved as `EV_EOL_<current_date>_docker.csv`, where `<current_date>` is dynamically determined.

---

### Expected Output (CSV):

#### Example File Name:
`EV_EOL_2025-01-24_docker.csv`

| repo_name               | package_type | package_name                                | version  | url                                                                                           | created_date              | license | secarch | artifactory_instance |
|-------------------------|--------------|---------------------------------------------|----------|-----------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| msartaz-train-docker-pr | Docker       | msartaz-train-docker-pr/ace-test/test-github-java | 1.0.18   | https://frigate.jfrog.io/artifactory/msartaz-train-docker-pr/ace-test/test-github-java/1.0.18/sha256.marker | 2024-05-01T17:45:42.3832 |         |         | frigate.jfrog.io     |
| msartaz-train-docker-pr | Docker       | msartaz-train-docker-pr/ace-test/test-github-java | 1.0.19   | https://frigate.jfrog.io/artifactory/msartaz-train-docker-pr/ace-test/test-github-java/1.0.19/sha256.marker | 2024-05-01T17:45:42.3832 |         |         | frigate.jfrog.io     |

---

### Steps to Run:

1. **Replace API Token**:
   - Replace `your_artifactory_api_token` with your actual JFrog Artifactory API token.

2. **Run the Script**:
   - Save the script as `docker_jfrog_metadata_extractor.py`.
   - Execute the script:
     ```bash
     python docker_jfrog_metadata_extractor.py
     ```

3. **Check the Output**:
   - The generated CSV file will contain only Docker repositories and related artifacts.

---

### Next Steps:

- **Test the Script**:
   - Run the script and check the output CSV file for Docker repositories.
  
- **Adjust for Additional Fields** (Optional):
   - If `license` or `secarch` fields are available, update the script to populate those dynamically.

Let me know how it works for you or if you need further refinements!
