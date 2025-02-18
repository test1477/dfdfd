Certainly! Here's the full updated script incorporating the changes to fetch both `manifest.json` and layer digests (`sha256:*`), along with additional debugging information:

```python
import requests
import csv
import os
import logging
import argparse
from datetime import datetime
from tqdm import tqdm

# Logging setup
logging.basicConfig(level=logging.DEBUG, format="%(asctime)s - %(levelname)s - %(message)s")

# Constants (Replace with your JFrog URL)
JFROG_URL = "https://your-jfrog-instance/artifactory"
ARTIFACTORY_API = f"{JFROG_URL}/api/storage"
REPOSITORIES_API = f"{JFROG_URL}/api/repositories"
BUILDS_API = f"{JFROG_URL}/api/build"
SEARCH_API = f"{JFROG_URL}/api/search/aql"
MAX_RETRIES = 3

def parse_arguments():
    parser = argparse.ArgumentParser(description="Fetch Docker image details from JFrog Artifactory.")
    parser.add_argument("--token", required=True, help="Artifactory API token")
    parser.add_argument("--output", required=True, help="Output directory for the CSV report")
    return parser.parse_args()

def get_headers(token):
    return {
        "Authorization": f"Bearer {token}",
        "Content-Type": "text/plain",
        "Accept": "application/json"
    }

def list_repositories(headers):
    try:
        response = requests.get(REPOSITORIES_API, headers=headers, verify=False)
        response.raise_for_status()
        return [repo for repo in response.json() if repo.get("packageType") == "Docker"]
    except requests.RequestException as e:
        logging.error(f"Error fetching repositories: {e}")
        return []

def get_build_info(image_path, version, headers):
    url = f"{BUILDS_API}/{image_path}/{version}"
    try:
        response = requests.get(url, headers=headers, verify=False)
        if response.status_code == 200:
            build_info = response.json()
            logging.debug(f"Build info for {image_path}/{version}: {build_info}")
            return build_info
        else:
            logging.warning(f"Failed to fetch build info for {image_path}/{version}: {response.status_code}")
    except requests.RequestException as e:
        logging.warning(f"Error fetching build info: {e}")
    return None

def get_eon_id_from_build_info(build_info):
    if not build_info:
        return ""
    
    eon_id = build_info.get("buildInfo", {}).get("env", {}).get("EON_ID")
    if eon_id:
        logging.debug(f"Found EON_ID in buildInfo.env: {eon_id}")
        return eon_id
    
    eon_id = build_info.get("buildInfo", {}).get("properties", {}).get("buildInfo.env.EON_ID")
    if eon_id:
        logging.debug(f"Found EON_ID in buildInfo.properties: {eon_id}")
        return eon_id
    
    logging.debug(f"EON_ID not found in build info: {build_info}")
    return ""

def get_artifacts_info(repo_name, headers):
    aql_query = f"""items.find(
        {{
            "repo": "{repo_name}",
            "$or": [
                {{"name": "manifest.json"}},
                {{"name": {{"$match": "sha256*"}}}},
                {{"name": {{"$match": "*.tar.gz"}}}}
            ]
        }}
    ).include("repo", "path", "name", "actual_sha1", "actual_md5")"""
    
    try:
        response = requests.post(SEARCH_API, data=aql_query, headers=headers, verify=False)
        response.raise_for_status()
        artifacts = response.json().get("results", [])
        logging.debug(f"Fetched artifacts: {artifacts}")
        return artifacts
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

def process_artifacts(repo_name, artifacts, headers):
    repo_details = []
    eon_id_cache = {}  # Cache to store EON_IDs for each image version

    for artifact in tqdm(artifacts, desc=f"Processing artifacts in {repo_name}"):
        path_parts = artifact.get("path", "").split('/')
        if len(path_parts) < 2:
            continue

        image_name = '/'.join(path_parts[:-1])
        tag = path_parts[-1]
        
        # Use cached EON_ID if available
        cache_key = f"{image_name}:{tag}"
        if cache_key not in eon_id_cache:
            build_info = get_build_info(f"{repo_name}/{image_name}", tag, headers)
            eon_id = get_eon_id_from_build_info(build_info)
            eon_id_cache[cache_key] = eon_id
        
        eon_id = eon_id_cache[cache_key]

        digest = artifact.get("actual_sha1", "")
        if not digest:
            continue

        formatted_digest = f"sha256:{digest}"

        repo_details.append({
            "Resource_Name": f"{image_name}:{tag}",
            "CSP": "placeholder",
            "Resource_Type": "Container Image",
            "Unique_ID": formatted_digest,
            "EON_ID": eon_id,
            "Digest": formatted_digest,
            "Registry": f"{repo_name}/{image_name}"
        })

    return repo_details

def process_repository(repo_name, headers):
    logging.info(f"Processing repository: {repo_name}")
    artifacts = get_artifacts_info(repo_name, headers)
    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")
    return process_artifacts(repo_name, artifacts, headers)

def save_to_csv(data, output_dir):
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = os.path.join(output_dir, f"EV_EOL_{current_date}.csv")
    headers = ['Resource_Name', 'CSP', 'Resource_Type', 'Unique_ID', 'EON_ID', 'Digest', 'Registry']
    
    os.makedirs(output_dir, exist_ok=True)
    
    with open(output_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    logging.info(f"Data has been written to {output_file}")

def main():
    args = parse_arguments()
    headers = get_headers(args.token)
    
    all_data = []
    
    repositories = list_repositories(headers)
    
    for repo in repositories:
        repo_data = process_repository(repo["key"], headers)
        all_data.extend(repo_data)

    save_to_csv(all_data, args.output)
    
    logging.info("Script execution completed")

if __name__ == "__main__":
    requests.packages.urllib3.disable_warnings()
    main()
```

This updated script includes the following key changes:

1. Modified `get_artifacts_info()` function to correctly fetch both `manifest.json` and layer digests (`sha256*`).
2. Added debug logging to print out the fetched artifacts.
3. Ensured that the EON_ID is retrieved and associated with all artifacts of a given image version.
4. Changed the script to use an empty string instead of "N/A" when the EON_ID is not found.

To use this script:

1. Save it as `jfrog_artifact_info.py` (or any preferred name).
2. Make sure you have the required libraries installed (`requests`, `tqdm`).
3. Replace the `JFROG_URL` constant at the top of the script with your actual JFrog Artifactory URL.
4. Run the script using:

   ```
   python jfrog_artifact_info.py --token YOUR_ARTIFACTORY_TOKEN --output /path/to/output/directory
   ```

   Replace `YOUR_ARTIFACTORY_TOKEN` with your actual JFrog Artifactory API token and specify the desired output directory.

This script should now correctly retrieve and process all relevant artifacts, including `manifest.json` and layer digests, while associating the correct EON_ID with each artifact. The debug logging will help you verify that all expected artifacts are being fetched and processed.

---
Answer from Perplexity: pplx.ai/share
