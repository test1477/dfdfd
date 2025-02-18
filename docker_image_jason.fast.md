I understand. Let's modify the script to retrieve the EON_ID for all artifacts associated with a Docker image, not just the manifest.json. We'll assume that all artifacts for a given image version share the same EON_ID. Here's the updated full script:

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
        return "N/A"
    
    eon_id = build_info.get("buildInfo", {}).get("env", {}).get("EON_ID")
    if eon_id:
        logging.debug(f"Found EON_ID in buildInfo.env: {eon_id}")
        return eon_id
    
    eon_id = build_info.get("buildInfo", {}).get("properties", {}).get("buildInfo.env.EON_ID")
    if eon_id:
        logging.debug(f"Found EON_ID in buildInfo.properties: {eon_id}")
        return eon_id
    
    logging.debug(f"EON_ID not found in build info: {build_info}")
    return "N/A"

def get_artifacts_info(repo_name, headers):
    aql_query = f"""items.find(
        {{
            "repo": "{repo_name}",
            "$or": [
                {{"name": {{"$match": "*.tar.gz"}}}},
                {{"name": "manifest.json"}},
                {{"name": {{"$match": "sha256:*"}}}}
            ]
        }}
    ).include("repo", "path", "name", "actual_sha1", "actual_md5")"""
    
    try:
        response = requests.post(SEARCH_API, data=aql_query, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("results", [])
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
        if cache_key in eon_id_cache:
            eon_id = eon_id_cache[cache_key]
        else:
            build_info = get_build_info(f"{repo_name}/{image_name}", tag, headers)
            eon_id = get_eon_id_from_build_info(build_info)
            eon_id_cache[cache_key] = eon_id

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

This updated script includes the following changes:

1. The `get_artifacts_info` function now retrieves all relevant artifacts, including manifest.json, *.tar.gz files, and sha256:* files.
2. The `process_artifacts` function processes all artifacts for a given image, using a cache to store EON_IDs for each image version to avoid redundant API calls.
3. The script now associates the same EON_ID with all artifacts of a given image version.

This should result in the EON_ID being retrieved and associated with all relevant artifacts, not just the manifest.json file. The script should now produce a more comprehensive output, including EON_IDs for all Docker image artifacts.

---
Answer from Perplexity: pplx.ai/share
