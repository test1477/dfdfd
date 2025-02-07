I apologize for the misunderstanding. You're right, we should process all artifacts, not just manifest.json files. Let's modify the script to handle all artifacts while still attempting to get the correct digest. Here's an updated version that should work for all artifacts:

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

def get_artifact_digest(repo_name, artifact_path):
    """Fetches the digest for an artifact."""
    url = f"{ARTIFACTORY_API}/{repo_name}/{artifact_path}"
    response = requests.get(url, headers=HEADERS, verify=False)
    if response.status_code == 200:
        checksums = response.json().get("checksums", {})
        return checksums.get("sha256") or checksums.get("sha1") or checksums.get("md5")
    return None

def extract_image_info(repo_name, artifact_path):
    """Extracts Docker image information from the artifact path."""
    path_parts = artifact_path.split('/')
    if len(path_parts) < 2:
        return None, None, None

    file_name = path_parts[-1]
    tag = path_parts[-2] if len(path_parts) > 2 else "latest"
    image_name = '/'.join(path_parts[:-2]) if len(path_parts) > 2 else path_parts[-2]

    return image_name, tag, file_name

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

            image_name, tag, file_name = extract_image_info(repo_name, artifact_path)
            if not image_name:
                continue  # Skip if we couldn't extract the necessary information

            digest = get_artifact_digest(repo_name, artifact_path)
            if not digest:
                continue  # Skip if we couldn't get the digest

            # Store data
            repo_details.append({
                "Docker Image Name": f"{image_name}:{tag}",
                "Resource Type": "Docker Image",
                "Unique ID": f"sha256:{digest}",
                "Digest": f"sha256:{digest}",
                "Repo Path": f"{repo_name}/{image_name}",
                "CSP": "placeholder"  # Add your logic to determine CSP here
            })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []

def save_to_csv(data, filename):
    """Saves extracted metadata to a CSV file."""
    headers = ['Docker Image Name', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path', 'CSP']
    
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

1. Processes all artifacts in Docker repositories, not just manifest.json files.
2. Attempts to get the digest (SHA256, SHA1, or MD5, in that order of preference) for each artifact.
3. Extracts image name and tag information from the artifact path.
4. Uses the artifact's digest for both the Unique ID and Digest fields.
5. Skips artifacts that don't have a valid digest.

This approach should provide a comprehensive list of all Docker artifacts in your JFrog Artifactory, including their correct digests. The script will attempt to use SHA256 digests when available, falling back to SHA1 or MD5 if necessary.

---
Answer from Perplexity: pplx.ai/share
