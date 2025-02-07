I apologize for the confusion. I understand now that you want the package name to be just the package itself, without the path, and the repo_path should be the path up to the package. Let's modify the script to achieve this. Here's the updated version:

```python
import requests
import csv
import os
import hashlib
import re

# Constants
JFROG_URL = "https://frigate.jfrog.io"
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"
ARTIFACTORY_TOKEN = "your_artifactory_token_here"

# Headers for API authentication
HEADERS = {
    "Authorization": f"Bearer {ARTIFACTORY_TOKEN}",
    "Accept": "application/json"
}

# Regex pattern to detect versions (e.g., 1.0.0, v2.3, 20240101)
VERSION_PATTERN = re.compile(r'(\d+\.\d+\.\d+|\d{4}\d{2}\d{2}|v\d+(\.\d+)?)')

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

def extract_package_info(artifact_path):
    """
    Extracts the package name and repo path from the artifact path.
    The package name is the last part before the version.
    The repo path is everything up to and including the package name.
    """
    path_parts = artifact_path.split('/')
    for i, part in enumerate(path_parts):
        if VERSION_PATTERN.match(part):
            package_name = path_parts[i-1] if i > 0 else ""
            repo_path = '/'.join(path_parts[:i])
            return package_name, repo_path
    return "", artifact_path  # If no version found, return empty package name and full path

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

            # Extract package name and repo_path
            package_name, repo_path = extract_package_info(artifact_path)

            # Construct the full artifact URL
            artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"

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

This updated script:

1. Extracts the `package_name` as just the package itself (the last part before the version).
2. Sets the `repo_path` as the path up to and including the package name.
3. Keeps the full artifact path in the URL.
4. Generates the `unique_id` and `digest` as before.

This should produce output matching your requirements:
```
repo_path           package_name    url                                                                          unique_id       digest
path/to/package     package         https://frigate.jfrog.io/artifactory/my-repo/path/to/package/1.0.0/file.tar   sha256_[hash]   sha256_[hash]
```

Make sure to replace `"your_artifactory_token_here"` with your actual JFrog Artifactory token before running the script.

---
Answer from Perplexity: pplx.ai/share
