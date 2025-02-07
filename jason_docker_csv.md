I understand the issue now. It seems that the digest field is sometimes populated with file names like "main.json" or "manifest.json" instead of the expected SHA digest. Let's modify the script to handle this and ensure we're getting the correct SHA digest for Docker images. Here's an updated version of the script that addresses this issue:

```python
import requests
import csv
import os
import json
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

def get_manifest_digest(repo_name, image_name, tag):
    """Fetches the manifest digest for a Docker image."""
    manifest_url = f"{ARTIFACTORY_API}/{repo_name}/{image_name}/{tag}/manifest.json"
    response = requests.get(manifest_url, headers=HEADERS, verify=False)
    if response.status_code == 200:
        manifest_data = response.json()
        return manifest_data.get("checksums", {}).get("sha256")
    return None

def extract_image_info(repo_name, artifact_path):
    """Extracts Docker image information from the artifact path."""
    path_parts = artifact_path.split('/')
    if len(path_parts) < 2:
        return None, None

    tag = path_parts[-2]  # Assuming the tag is always the second-to-last part
    image_name = '/'.join(path_parts[:-2])  # Everything before the tag

    return image_name, tag

def process_repository(repo_name):
    """Processes all artifacts in a repository."""
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "").lstrip("/")

            # Skip .jfrog cache folder and non-manifest files
            if ".jfrog" in artifact_path or not artifact_path.endswith("manifest.json"):
                continue

            image_name, tag = extract_image_info(repo_name, artifact_path)
            if not image_name or not tag:
                continue  # Skip if we couldn't extract the necessary information

            digest = get_manifest_digest(repo_name, image_name, tag)
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

1. Focuses on processing manifest.json files, which contain the correct digest information for Docker images.
2. Introduces a new function `get_manifest_digest` to fetch the SHA256 digest from the manifest file.
3. Uses the SHA256 digest from the manifest for both the Unique ID and Digest fields.
4. Skips artifacts that don't have a valid manifest or digest.

This approach should resolve the issue of incorrect digest values and ensure that you're getting the proper SHA256 digest for each Docker image. The Unique ID and Digest fields will now contain the actual SHA256 digest prefixed with "sha256:", as is standard for Docker image digests.

---
Answer from Perplexity: pplx.ai/share
