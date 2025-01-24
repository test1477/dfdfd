To retrieve **all artifact versions** and their corresponding **package names** under a repository in JFrog Artifactory, we need to query the **repository contents recursively**. The JFrog REST API provides the `api/storage/<repo_name>` endpoint, which can be used with the `list` parameter to fetch the full directory tree. This allows us to navigate through all paths and extract version-specific artifacts.

Here’s how we can enhance the script to list all artifacts and extract the version and package name for each artifact:

---

### Enhanced Script for Recursive Artifact Retrieval
This script:
1. Queries the repository tree recursively to fetch all artifacts.
2. Extracts version information from the directory structure.
3. Handles repositories with multiple versions of the same package.

```python
import requests
import csv
import warnings

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# JFrog Artifactory instance details
JFROG_URL = 'https://frigate.jfrog.io/artifactory'  # Correct base URL
ARTIFACTORY_TOKEN = 'your_artifactory_api_token'  # Replace with your actual API token

# Headers for the request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

# Function to fetch repositories from JFrog Artifactory
def fetch_repositories():
    url = f"{JFROG_URL}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()  # Return JSON response if successful
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

# Function to fetch all artifacts recursively from a repository
def fetch_artifacts_recursive(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"  # Recursive fetch
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()  # Return JSON response if successful
    else:
        print(f"Failed to fetch artifacts for {repo_name}: {response.status_code}")
        return {}

# Function to infer version and package name from artifact path
def extract_version_and_package(path):
    segments = path.strip("/").split("/")
    if len(segments) >= 2:
        package_name = segments[0]  # Assume the first segment is the package name
        version = segments[1]  # Assume the second segment is the version
        return package_name, version
    return "N/A", "N/A"

# Function to extract detailed artifact information
def extract_repo_details(repositories):
    repo_details = []

    for repo in repositories:
        repo_name = repo.get("key", "N/A")
        repo_type = repo.get("type", "N/A")  # Local, remote, or virtual
        package_type = repo.get("packageType", "N/A")  # npm, Docker, etc.

        # Fetch artifact details for the repository
        artifacts_data = fetch_artifacts_recursive(repo_name)

        if "files" in artifacts_data:  # Check if the repository has artifacts
            for artifact in artifacts_data["files"]:
                artifact_path = artifact.get("uri", "N/A")
                artifact_url = f"{JFROG_URL}/{repo_name}{artifact_path}"
                created_date = artifact.get("lastModified", "N/A")
                size = artifact.get("size", "N/A")
                package_name, version = extract_version_and_package(artifact_path)

                repo_details.append([
                    repo_name, repo_type, package_type, package_name, version,
                    artifact_url, created_date, "N/A", "frigate.jfrog.io"
                ])
        else:
            # If no artifacts are found, append basic repository information
            repo_details.append([
                repo_name, repo_type, package_type, "N/A", "N/A",
                f"{JFROG_URL}/{repo_name}", "N/A", "N/A", "frigate.jfrog.io"
            ])

    return repo_details

# Function to save the extracted data into a CSV file
def save_to_csv(data, filename='jfrog_repositories.csv'):
    headers = ['Repo Name', 'Repo Type', 'Package Type', 'Package Name', 'Version', 'URL', 'Creation Date', 'License', 'Artifact Instance']
    
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerow(headers)
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

# Main function to fetch and process the repositories data
def main():
    repositories = fetch_repositories()

    if repositories:
        repo_details = extract_repo_details(repositories)
        save_to_csv(repo_details)
    else:
        print("No repository data found.")

if __name__ == '__main__':
    main()
```

---

### Key Updates:
1. **Recursive Artifact Retrieval**:
   - The script uses the `?list&deep=1` query parameter in the `api/storage/<repo_name>` endpoint to fetch all artifact paths recursively.

2. **Version and Package Name Extraction**:
   - The `extract_version_and_package` function assumes the directory structure follows a standard pattern like:
     ```
     <repo_name>/<package_name>/<version>/<artifact_name>
     ```
   - It extracts the package name and version from the first two segments of the path.

3. **Enhanced Artifact Details**:
   - The script now includes:
     - `artifact_url`: Full URL of the artifact.
     - `created_date`: The last modified timestamp of the artifact.
     - `size`: (Optional) Size of the artifact (currently unused in CSV).

---

### Example Output (CSV):

For a repository `npm-repo` with artifacts structured as:

```plaintext
npm-repo/
    lodash/
        4.17.21/
            lodash-4.17.21.tgz
    react/
        17.0.2/
            react-17.0.2.tgz
```

The output will look like this:

| Repo Name  | Repo Type | Package Type | Package Name | Version  | URL                                             | Creation Date | License | Artifact Instance   |
|------------|-----------|--------------|--------------|----------|-------------------------------------------------|---------------|---------|---------------------|
| npm-repo   | local     | npm          | lodash       | 4.17.21  | https://frigate.jfrog.io/npm-repo/lodash/4.17.21/lodash-4.17.21.tgz | 2024-01-23T12:34:56 | N/A     | frigate.jfrog.io    |
| npm-repo   | local     | npm          | react        | 17.0.2   | https://frigate.jfrog.io/npm-repo/react/17.0.2/react-17.0.2.tgz     | 2024-01-23T12:34:56 | N/A     | frigate.jfrog.io    |

---

### Notes:
1. **Standard Directory Structure**:
   - This script assumes a standard directory structure. If your Artifactory uses a different layout, modify `extract_version_and_package`.

2. **License Field**:
   - Currently hardcoded as `"N/A"`. You can extend the script to fetch license details from Artifactory metadata (if available).

3. **Pagination Handling**:
   - If there are many artifacts, the API may paginate results. Let me know if pagination is required, and I’ll help you enhance the script.

---

### Next Steps:
1. Test the script and verify the output CSV.
2. Check if the directory structure and inferred versions align with your expectations.
3. Let me know if you need further refinements or additional features!
