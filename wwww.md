To modify the script to fetch artifacts from all repositories in JFrog Artifactory, you need to first retrieve a list of all repositories and then iterate through them. Here's an updated version of the script that accomplishes this:

```python
import requests
import csv
import warnings
from urllib.parse import urljoin

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# JFrog Artifactory instance details
JFROG_URL = 'https://frigate.jfrog.io/artifactory'
ARTIFACTORY_TOKEN = 'your_artifactory_api_token'

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

def get_all_repositories():
    url = f"{JFROG_URL}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return [repo['key'] for repo in response.json()]
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

def fetch_metadata(repo_name, artifact_path):
    url = f"{JFROG_URL}/api/storage/{repo_name}{artifact_path}"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch metadata for {repo_name}{artifact_path}: {response.status_code}")
        return {}

def parse_artifact_details(repo_name, artifact_path):
    segments = artifact_path.strip("/").split("/")
    if len(segments) >= 2:
        package_name = f"{repo_name}/{'/'.join(segments[:-2])}"
        version = segments[-2]
        return package_name, version
    return f"{repo_name}/N/A", "N/A"

def process_repository(repo_name):
    artifact_list = list_artifacts(repo_name)
    repo_details = []
    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        metadata = fetch_metadata(repo_name, artifact_path)
        package_name, version = parse_artifact_details(repo_name, artifact_path)
        repo_details.append({
            "repo_name": metadata.get("repo", repo_name),
            "package_type": "Docker",  # You may need to determine this dynamically
            "package_name": package_name,
            "version": version,
            "url": metadata.get("downloadUri", "N/A"),
            "created_date": metadata.get("created", "N/A"),
            "artifactory_instance": "frigate.jfrog.io"
        })
    return repo_details

def save_to_csv(data, filename='jfrog_artifact_metadata.csv'):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'artifactory_instance']
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    print(f"Data has been written to {filename}")

def main():
    repositories = get_all_repositories()
    all_repo_details = []
    for repo_name in repositories:
        print(f"Processing repository: {repo_name}")
        repo_details = process_repository(repo_name)
        all_repo_details.extend(repo_details)
    save_to_csv(all_repo_details)

if __name__ == '__main__':
    main()
```

This updated script includes the following changes:

1. A new function `get_all_repositories()` has been added to fetch the list of all repositories from JFrog Artifactory[1].

2. The `main()` function now uses `get_all_repositories()` to get a list of all repositories instead of using a hardcoded list[1].

3. The script iterates through all repositories, processing each one and collecting metadata for all artifacts[1].

4. The `parse_artifact_details()` function remains the same, extracting package name and version from the artifact path.

5. The script still uses the same CSV output format, saving data for all repositories in a single file.

Remember to replace `'your_artifactory_api_token'` with your actual JFrog Artifactory API token[1]. Also, make sure you have the necessary permissions to access all repositories and their contents.

This script will now fetch metadata for artifacts from all repositories in your JFrog Artifactory instance, providing a comprehensive overview of your artifacts across all repositories[1][3].

Citations:
[1] https://stackoverflow.com/questions/54200143/artifcatory-rest-api-to-list-all-the-repositories/54207176
[2] https://stackoverflow.com/questions/27981545/suppress-insecurerequestwarning-unverified-https-request-is-being-made-in-pytho/44850849
[3] https://stackoverflow.com/questions/72318187/get-all-repositories-of-two-package-types-via-artifactory-api/72335821
[4] https://jfrog.com/help/r/jfrog-rest-apis/get-certificates
[5] https://releases.jfrog.io/artifactory/oss-releases-local/org/artifactory/artifactory-papi/%5BRELEASE%5D/artifactory-papi-%5BRELEASE%5D-javadoc.jar!/org/artifactory/repo/Repositories.html
[6] https://jfrog.com/help/r/artifactory-how-to-resolve-an-unable-to-find-valid-certification-path-to-requested-target-error/you-may-encounter-an-issue-when-trying-to-establish-a-trusted-ssl-connection-from-artifactory
[7] https://jfrog.com/help/r/jfrog-rest-apis/get-folder-info
[8] https://jfrog.com/help/r/artifactory-how-to-resolve-an-unable-to-find-valid-certification-path-to-requested-target-error
