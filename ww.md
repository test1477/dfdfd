Certainly! Here's the full updated script incorporating all the changes we've discussed:

```python
import requests
import csv
import os
import concurrent.futures
import logging
from tqdm import tqdm

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Constants
JFROG_URL = "https://frigate.jfrog.io"
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"
ARTIFACTORY_TOKEN = "your_artifactory_token_here"
MAX_WORKERS = 10
MAX_RETRIES = 3

# Headers for API authentication
HEADERS = {
    "Authorization": f"Bearer {ARTIFACTORY_TOKEN}",
    "Accept": "application/json"
}

def list_repositories():
    url = f"{JFROG_URL}/artifactory/api/repositories"
    response = requests.get(url, headers=HEADERS, verify=False)
    response.raise_for_status()
    return [repo for repo in response.json() if repo.get("packageType") == "Docker"]

def list_artifacts(repo_name):
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1&listFolders=0"
    response = requests.get(url, headers=HEADERS, verify=False)
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        logging.error(f"Error fetching artifacts for {repo_name}: {response.text}")
        return []

def get_artifact_info(repo_name, artifact):
    artifact_path = artifact.get("uri", "").lstrip("/")
    if ".jfrog" in artifact_path:
        return None

    url = f"{ARTIFACTORY_API}/{repo_name}/{artifact_path}"
    for _ in range(MAX_RETRIES):
        try:
            response = requests.get(url, headers=HEADERS, verify=False, timeout=10)
            response.raise_for_status()
            data = response.json()
            digest = data.get("checksums", {}).get("sha256")
            if not digest:
                return None

            path_parts = artifact_path.split('/')
            
            # Handle cases where path_parts might be too short
            if len(path_parts) < 2:
                logging.warning(f"Unexpected path structure for {artifact_path}")
                return None

            tag = path_parts[-2] if len(path_parts) > 2 else "latest"
            image_name = '/'.join(path_parts[:-2]) if len(path_parts) > 2 else path_parts[0]

            return {
                "Docker Image Name": f"{image_name}:{tag}",
                "Resource Type": "Docker Image",
                "Unique ID": f"sha256:{digest}",
                "Digest": f"sha256:{digest}",
                "Repo Path": f"{repo_name}/{image_name}",
                "CSP": "placeholder"
            }
        except requests.RequestException as e:
            logging.warning(f"Error fetching {url}: {e}. Retrying...")
        except IndexError as e:
            logging.error(f"IndexError processing {artifact_path}: {e}")
            return None
    
    logging.error(f"Failed to fetch {url} after {MAX_RETRIES} attempts")
    return None

def process_repository(repo_name):
    logging.info(f"Processing repository: {repo_name}")
    artifacts = list_artifacts(repo_name)
    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")

    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        future_to_artifact = {executor.submit(get_artifact_info, repo_name, artifact): artifact for artifact in artifacts}
        repo_details = []
        for future in tqdm(concurrent.futures.as_completed(future_to_artifact), total=len(artifacts), desc=f"Processing {repo_name}"):
            try:
                result = future.result()
                if result:
                    repo_details.append(result)
            except Exception as e:
                logging.error(f"Error processing artifact in {repo_name}: {e}")

    logging.info(f"Processed {len(repo_details)} valid artifacts in {repo_name}")
    return repo_details

def save_to_csv(data, filename):
    headers = ['Docker Image Name', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path', 'CSP']
    os.makedirs(os.path.dirname(filename), exist_ok=True)
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    logging.info(f"Data has been written to {filename}")

def main():
    repositories = list_repositories()
    logging.info(f"Found {len(repositories)} Docker repositories")

    all_data = []
    for repo in repositories:
        repo_data = process_repository(repo["key"])
        all_data.extend(repo_data)

    save_to_csv(all_data, "c:/sre/jfrog_report.csv")
    logging.info("Script execution completed")

if __name__ == "__main__":
    requests.packages.urllib3.disable_warnings()
    main()
```

This script includes all the improvements and error handling we've discussed. It should now:

1. Handle potential IndexErrors when processing artifact paths.
2. Provide more detailed logging information.
3. Skip artifacts that can't be processed correctly.
4. Continue processing even if some artifacts or repositories encounter errors.

To use this script:

1. Install the required packages: `pip install requests tqdm`
2. Replace `"your_artifactory_token_here"` with your actual JFrog Artifactory token.
3. Adjust `MAX_WORKERS` if needed, based on your system's capabilities and API rate limits.
4. Ensure you have write permissions for the output file path (c:/sre/jfrog_report.csv).

This version should be more robust and provide better visibility into the script's progress and any issues that may occur during execution.

---
Answer from Perplexity: pplx.ai/share
