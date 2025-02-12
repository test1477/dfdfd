Here's the modified script for a single repository that uses the provided values:

```python
import requests
import csv
import os
import logging
from datetime import datetime
from tqdm import tqdm

logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

JFROG_URL = "http://your-local-artifactory-url:8081/artifactory"
ARTIFACTORY_API = f"{JFROG_URL}/api/storage"
BUILDS_API = f"{JFROG_URL}/api/build"
MAX_RETRIES = 3

def get_headers(token):
    return {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    }

def list_artifacts(repo_name, headers):
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1&listFolders=0"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

def get_artifact_info(repo_name, artifact, headers):
    artifact_path = artifact.get("uri", "").lstrip("/")
    if ".jfrog" in artifact_path:
        return None

    url = f"{ARTIFACTORY_API}/{repo_name}/{artifact_path}"
    for _ in range(MAX_RETRIES):
        try:
            response = requests.get(url, headers=headers, verify=False)
            response.raise_for_status()
            data = response.json()
            digest = data.get("checksums", {}).get("sha256")
            if not digest:
                return None

            path_parts = artifact_path.split('/')
            if len(path_parts) < 2:
                logging.warning(f"Unexpected path structure for {artifact_path}")
                return None

            tag = path_parts[-2] if len(path_parts) > 2 else "latest"
            image_name = '/'.join(path_parts[:-2]) if len(path_parts) > 2 else path_parts[0]

            eon_id = data.get("properties", {}).get("EON_ID", "N/A")
            formatted_digest = f"sha256:{digest}" if not digest.startswith("sha256:") else digest

            return {
                "Resource_Name": f"{image_name}:{tag}",
                "CSP": "placeholder",
                "Resource_Type": "Container Image",
                "Unique_ID": formatted_digest,
                "EON_ID": eon_id,
                "Digest": formatted_digest,
                "Registry": f"{repo_name}/{image_name}"
            }
        except requests.RequestException as e:
            logging.warning(f"Error fetching {url}: {e}. Retrying...")

    logging.error(f"Failed to fetch {url} after {MAX_RETRIES} attempts")
    return None

def process_repository(repo_name, headers):
    logging.info(f"Processing repository: {repo_name}")
    artifacts = list_artifacts(repo_name, headers)
    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")

    repo_details = []
    for artifact in tqdm(artifacts, desc=f"Processing artifacts in {repo_name}"):
        result = get_artifact_info(repo_name, artifact, headers)
        if result:
            repo_details.append(result)

    logging.info(f"Processed {len(repo_details)} valid artifacts in {repo_name}")
    return repo_details

def save_to_csv(data, output_file):
    headers = ['Resource_Name', 'CSP', 'Resource_Type', 'Unique_ID', 'EON_ID', 'Digest', 'Registry']
    
    with open(output_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    logging.info(f"Data has been written to {output_file}")

if __name__ == "__main__":
    requests.packages.urllib3.disable_warnings()
    
    token = "YOUR_API_KEY"
    repo_name = "YOUR_REPO_NAME"
    output_file = f"EV_EOL_2025-02-12.csv"

    headers = get_headers(token)
    repo_data = process_repository(repo_name, headers)
    save_to_csv(repo_data, output_file)
    logging.info("Script execution completed")
```

To use this script:

1. Replace `YOUR_API_KEY` with your actual JFrog Artifactory API key.
2. Replace `YOUR_REPO_NAME` with the name of the repository you want to target.
3. Update the `JFROG_URL` constant with your local JFrog Artifactory URL.
4. Run the script in the current folder.

The script will generate a CSV file named `EV_EOL_2025-02-12.csv` in the current directory with the processed data[1][3].

Citations:
[1] https://jfrog.com/integrations/python-repository/
[2] https://docs.mend.io/wsk/python-with-jfrog-artifactory-host-rule-implementa
[3] https://devopshq.github.io/artifactory/
[4] https://jfrog.com/help/r/jfrog-artifactory-documentation/upload-authenticated-pypi-packages-to-jfrog-artifactory
[5] https://jfrog.com/help/r/jfrog-artifactory-documentation/pypi-repositories
[6] https://jfrog.com/screencast/setting-up-pypi-repository-in-minutes-with-jfrog-artifactory/
[7] https://stackoverflow.com/questions/27270793/deploy-to-artifactory-via-python-script
[8] https://pypi.org/project/PyArtifactory/

---
Answer from Perplexity: pplx.ai/share
