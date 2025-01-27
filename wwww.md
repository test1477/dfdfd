To include additional fields like `license` and `secarch` in the output, as well as save the file with the name format `EV_EOL_<current_date>.csv`, we can make the following updates:

---

### Updated Script

This version:
1. **Includes `license` and `secarch`**:
   - These fields can be placeholders (`"N/A"`) if they are unavailable in the metadata.
2. **Dynamically Saves the Output File**:
   - The file is saved as `EV_EOL_<current_date>.csv`.

---

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
        created_date = artifact.get("lastModified", "N/A")

        # Parse package name and version from artifact path
        segments = artifact_path.strip("/").split("/")
        if len(segments) >= 2:
            package_name = f"{repo_name}/{'/'.join(segments[:-2])}"  # Up to the version folder
            version = segments[-2]  # Second-to-last segment
        else:
            package_name = repo_name
            version = "N/A"

        # Append details
        repo_details.append({
            "repo_name": repo_name,
            "package_type": package_type,
            "package_name": package_name,
            "version": version,
            "url": artifact_url,
            "created_date": created_date,
            "license": "N/A",  # Placeholder, adjust if license data is available
            "secarch": "N/A",  # Placeholder, adjust if secarch data is available
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
        package_type = repo.get("packageType", "N/A")  # Dynamically fetch the package type
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

### Key Updates:
1. **`license` and `secarch` Fields**:
   - Added placeholders for `license` and `secarch` in the output.
   - If data for these fields is available in the artifact metadata, the script can be enhanced to fetch and populate them dynamically.

2. **Dynamic File Naming**:
   - The output file name is now dynamically generated as `EV_EOL_<current_date>.csv`, where `<current_date>` is in the format `YYYY-MM-DD`.

3. **Efficient Metadata Handling**:
   - Uses batch metadata processing via the `?list&deep=1` API endpoint for faster execution.

---

### Output Example:

#### Input Repository Structure:
```plaintext
msartaz-train-docker-pr/
    ace-test/
        test-github-java/
            1.0.18/
                sha256.marker
npm-ppa-virtual/
    ppa-fip-security-documentation-client/
        2.3.0/
            client-2.3.0.tgz
```

#### Generated CSV:
File Name: `EV_EOL_2025-01-24.csv`

| repo_name               | package_type | package_name                                | version  | url                                                                                           | created_date              | license | secarch | artifactory_instance |
|-------------------------|--------------|---------------------------------------------|----------|-----------------------------------------------------------------------------------------------|---------------------------|---------|---------|-----------------------|
| msartaz-train-docker-pr | Docker       | msartaz-train-docker-pr/ace-test/test-github-java | 1.0.18   | https://frigate.jfrog.io/artifactory/msartaz-train-docker-pr/ace-test/test-github-java/1.0.18/sha256.marker | 2024-05-01T17:45:42.3832 | N/A     | N/A     | frigate.jfrog.io     |
| npm-ppa-virtual         | npm          | npm-ppa-virtual/ppa-fip-security-documentation-client | 2.3.0    | https://frigate.jfrog.io/artifactory/npm-ppa-virtual/ppa-fip-security-documentation-client/2.3.0/client-2.3.0.tgz | 2024-01-15T14:20:50.1234 | N/A     | N/A     | frigate.jfrog.io     |

---

### Next Steps:
1. **Run the Script**:
   - Replace `your_artifactory_api_token` with your JFrog API token.
   - Execute the script and check the generated file.

2. **Populate `license` and `secarch`**:
   - If `license` or `secarch` is part of the metadata, we can adjust the script to include it.

Let me know if you need further adjustments!
