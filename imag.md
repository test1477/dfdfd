Certainly! I understand you want to use `urlparse` to extract the package name and repo path. Here's the updated script incorporating this approach:

```python
import requests
import csv
import os
import hashlib
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
    """
    Extracts the package name and repo path from the URL.
    The package name is the second-to-last segment.
    The repo path is everything up to but not including the package name and version.
    """
    parsed_url = urlparse(url)
    path_segments = parsed_url.path.strip("/").split("/")
    
    package_name = path_segments[-2] if len(path_segments) > 2 else ""
    repo_path = "/".join(path_segments[:-2])
    
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
                "digest": unique_id
            })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []

def save_to_csv(data, filename):
    """Saves extracted metadata to a CSV file."""
    headers = ['repo_path', 'package_name', 'url', 'unique_id', 'digest']
    
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

This script now uses `urlparse` to extract the package name and repo path. The `extract_package_info` function has been updated to use this method. This approach should correctly handle the URL structure and extract the package name and repo path as per your requirements.

---
Answer from Perplexity: pplx.ai/share
