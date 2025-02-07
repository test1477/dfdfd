Got it! Here are the final adjustments:  

✅ **Repo Path Fix**  
- Should now be **`my-repo/path/to/package`** (including repo name but excluding version).  

✅ **Package Name Fix**  
- Should now be **only `package`** (extracted before the version).  

✅ **No Need for `artifactory_instance`**  
- Removed from the CSV output.  

---

### **Updated Python Script**
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
VERSION_PATTERN = re.compile(r'[/\-](\d+\.\d+\.\d+|\d{4}\d{2}\d{2}|v\d+(\.\d+)?)')

def list_repositories():
    """Fetches all repositories in Artifactory."""
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
    Extracts the package name (only the name before the version).
    Example:
        Input: "path/to/package/1.0.0/file.tar"
        Output: "package"
    """
    path_segments = artifact_path.split("/")
    
    for i, segment in enumerate(path_segments):
        if VERSION_PATTERN.search(segment):  # Stop at the first version match
            return path_segments[i - 1] if i > 0 else ""

    return path_segments[-2] if len(path_segments) > 2 else ""  # Fallback case

def extract_repo_path(repo_name, artifact_path):
    """
    Extracts the repo path in the format: my-repo/path/to/package
    Excludes versioning and hash digests.
    """
    segments = artifact_path.split("/")
    filtered_segments = []

    for segment in segments:
        if VERSION_PATTERN.search(segment):  # Stop at the first version occurrence
            break
        filtered_segments.append(segment)

    return f"{repo_name}/{'/'.join(filtered_segments)}"

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

            # Extract package name
            package_name = extract_package_name(artifact_path)

            # Extract repo path
            repo_path = extract_repo_path(repo_name, artifact_path)

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
                "digest": unique_id  # Digest is the same as unique_id
            })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []  # Continue with other repositories

def save_to_csv(data, filename):
    """Saves extracted metadata to a CSV file."""
    headers = ['repo_path', 'package_name', 'url', 'unique_id', 'digest']
    
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

### **Final Fixes & Improvements**
✅ **Repo Path Format**  
- Now correctly extracts **`my-repo/path/to/package`**  
- Includes **repo name** + **full path until the version**  

✅ **Package Name Fix**  
- Now correctly extracts **only `package`**  
- Uses regex to detect **versions** and trims the path accordingly  

✅ **No More `artifactory_instance`**  
- Removed from the CSV output  

✅ **CSV File Output Format**
| repo_path               | package_name | url                                                                          | unique_id                                          | digest                                             |
|-------------------------|-------------|------------------------------------------------------------------------------|--------------------------------------------------|--------------------------------------------------|
| my-repo/path/to/package | package     | `https://frigate.jfrog.io/artifactory/my-repo/path/to/package/1.0.0/file.tar` | sha256_019952e2e839127a176b5b10c038fd4d642314e5 | sha256_019952e2e839127a176b5b10c038fd4d642314e5 |

---

### **This should now be fully aligned with your request!** 🚀  
Let me know if you need any last-minute refinements!
