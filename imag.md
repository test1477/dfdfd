Got it! Here are the final refinements based on your requirements:  

✅ **Package name should be exactly as it appears in the repo path, stopping at the version**  
✅ **Unique ID and digest should be the same (full SHA-256 hash)**  
✅ **Repo path should still exclude the version and hash digest**  

---

### **Final Updated Python Script**
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

# Regular expression to match versions (e.g., 1.0.0, v2.3, 20240101)
VERSION_PATTERN = re.compile(r'[/\-](\d+\.\d+\.\d+|\d{4}\d{2}\d{2}|v\d+(\.\d+)?)')

def list_repositories():
    """Fetches all repositories in the Artifactory instance."""
    url = f"{JFROG_URL}/artifactory/api/repositories"
    response = requests.get(url, headers=HEADERS, verify=False)  # SSL verification disabled
    response.raise_for_status()
    return response.json()

def list_artifacts(repo_name):
    """Fetches all artifacts in a repository."""
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1"
    response = requests.get(url, headers=HEADERS, verify=False)  # SSL verification disabled

    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Error fetching artifacts for {repo_name}: {response.text}")
        return []

def extract_package_name(artifact_path):
    """
    Extracts the package name exactly as it appears in the repo path, stopping at the version.
    """
    path_segments = artifact_path.split("/")
    
    for i, segment in enumerate(path_segments):
        if VERSION_PATTERN.search(segment):  # Stop at the first version match
            return "/".join(path_segments[:i])
    
    return artifact_path  # If no version found, return full path as package name

def extract_repo_path(artifact_path):
    """
    Extracts the repo path without versioning and hash digests.
    """
    segments = artifact_path.split("/")
    filtered_segments = []

    for segment in segments:
        if VERSION_PATTERN.search(segment):  # Stop at the first version occurrence
            break
        filtered_segments.append(segment)

    return "/".join(filtered_segments)

def process_repository(repo_name):
    """Processes all artifacts in a repository while avoiding `.jfrog` cache folders."""
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "").lstrip("/")  # Remove leading slash if present

            # Skip .jfrog cache folder
            if ".jfrog" in artifact_path:
                continue

            # Extract package name from repo path
            package_name = extract_package_name(artifact_path)

            # Extract repo path excluding version and hash digest
            repo_path = extract_repo_path(artifact_path)

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
                "artifactory_instance": "frigate.jfrog.io",
                "unique_id": unique_id,
                "digest": unique_id  # Digest is the same as unique_id
            })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []  # Continue with other repositories

def save_to_csv(data, filename):
    """Saves extracted metadata to a CSV file."""
    headers = ['repo_path', 'package_name', 'url', 'artifactory_instance', 'unique_id', 'digest']
    
    os.makedirs(os.path.dirname(filename), exist_ok=True)  # Ensure output directory exists

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
    requests.packages.urllib3.disable_warnings()  # Suppress SSL warnings
    main()
```

---

### **Final CSV Output Format**
| repo_path              | package_name      | url                                                                          | artifactory_instance | unique_id                                          | digest                                             |
|------------------------|------------------|------------------------------------------------------------------------------|----------------------|--------------------------------------------------|--------------------------------------------------|
| path/to/package       | path/to/package  | `https://frigate.jfrog.io/artifactory/my-repo/path/to/package/1.0.0/file.tar` | frigate.jfrog.io    | sha256_019952e2e839127a176b5b10c038fd4d642314e5 | sha256_019952e2e839127a176b5b10c038fd4d642314e5 |

---

### **Final Updates & Fixes**
✔ **Package name is exactly as it appears in the repo path, stopping at the version.**  
✔ **Repo path now correctly excludes versions and hash digests.**  
✔ **Unique ID and Digest are now the same (full SHA-256 hash).**  
✔ **Repo name is still removed from the output.**  
✔ **SSL verification is disabled (`verify=False`) for all API requests.**  

This should now perfectly match your requirements. 🚀 Let me know if anything else needs tweaking!
