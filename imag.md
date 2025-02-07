Certainly! Here's the full script with the updated `extract_package_info` function and a placeholder for CSP (Cloud Service Provider). I've added a 'csp' field to the output, which you can populate as needed:

```python
import requests
import csv
import os
import hashlib
import re
from urllib.parse import urlparse

# Constants
JFROG_URL = "https://frigate.jfrog.io"
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"
ARTIFACTORY_TOKEN = "your_artifactory_token_here"

# Headers for API authentication
HEADERS = {
    "Authorization": f"Bearer {ARTIFACTORY_TOKEN}",
    "Accept": "application/json"
}

def list_repositories():
    """Fetches all repositories in Artifactory."""
    url = f"{JFROG_URL}/artifactory/api/repositories"
    response = requests.get(url, headers=HEADERS, verify=False)
    response.raise_for_status()
    return response.json()

def list_artifacts(repo_name):
    """Fetches all artifacts in a repository."""
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1"
    response = requests.get(url, headers=HEADERS, verify=False)

    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Error fetching artifacts for {repo_name}: {response.text}")
        return []

def extract_package_info(url):
    parsed_url = urlparse(url)
    path_segments = parsed_url.path.strip("/").split("/")
    
    # Find the index of the first segment that looks like a version number
    version_index = next((i for i, segment in enumerate(path_segments) if re.match(r'\d+(\.\d+)*', segment)), len(path_segments))
    
    # Package name is everything up to but not including the version
    package_name = "/".join(path_segments[:version_index])
    repo_path = "/".join(path_segments[:version_index])
    
    return package_name, repo_path

def process_repository(repo_name):
    """Processes all artifacts in a repository."""
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "").lstrip("/")

            # Skip .jfrog cache folder
            if ".jfrog" in artifact_path:
                continue

            # Construct the full artifact URL
            artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"

            # Extract package name and repo_path
            package_name, repo_path = extract_package_info(artifact_url)

            # Generate unique ID using SHA-256 hash
            sha256_hash = hashlib.sha256(artifact_url.encode()).hexdigest()
            unique_id = f"sha256_{sha256_hash}"

            # Store data
            repo_details.append({
                "repo_path": repo_path,
                "package_name": package_name,
                "url": artifact_url,
                "unique_id": unique_id,
                "digest": unique_id,
                "csp": "placeholder"  # Add your logic to determine CSP here
            })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []

def save_to_csv(data, filename):
    """Saves extracted metadata to a CSV file."""
    headers = ['repo_path', 'package_name', 'url', 'unique_id', 'digest', 'csp']
    
    os.makedirs(os.path.dirname(filename), exist_ok=True)

    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

def main():
    """Main function to process repositories and generate the CSV report."""
    repositories = list_repositories()
    all_data = []

    for repo in repositories:
        repo_name = repo.get("key", "")
        package_type = repo.get("packageType", "")

        # Process only Docker repositories
        if package_type == "Docker":
            print(f"Processing repository: {repo_name}")
            repo_data = process_repository(repo_name)
            all_data.extend(repo_data)

    # Save data to CSV
    save_to_csv(all_data, "c:/sre/jfrog_report.csv")

if __name__ == "__main__":
    requests.packages.urllib3.disable_warnings()
    main()
```

This script now:
1. Correctly extracts the package name without including the version.
2. Sets the repo_path to be the same as the package name.
3. Includes a 'csp' field in the output, currently set to "placeholder".

To use this script:
1. Replace `"your_artifactory_token_here"` with your actual JFrog Artifactory token.
2. If needed, implement logic to determine the CSP for each artifact. You can do this by modifying the `process_repository` function where the "csp" field is currently set to "placeholder".
3. Ensure you have the necessary permissions to access the repositories and artifacts in your JFrog Artifactory instance.

This script should now produce output that matches your requirements, including the correct package name extraction and a placeholder for CSP information.

---
Answer from Perplexity: pplx.ai/share
