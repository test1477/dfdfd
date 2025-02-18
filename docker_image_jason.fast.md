Certainly! Here's a full script that incorporates the logic to retrieve the EON_ID based on the Docker image push process you described. This script will attempt to fetch the EON_ID from the build info associated with each artifact:

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
JFROG_URL = "https://parametric.jfrog.io/artifactory"
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

def get_manifest_info(repo_name, image_name, tag, headers):
    aql_query = f"""items.find(
        {{
            "repo": "{repo_name}",
            "path": {{
                "$match": "{image_name}/*"
            }},
            "name": "manifest.json"
        }}
    ).include("repo", "path", "name", "actual_sha1", "actual_md5")"""
    
    try:
        response = requests.post(SEARCH_API, data=aql_query, headers=headers, verify=False)
        response.raise_for_status()
        results = response.json().get("results", [])
        for result in results:
            if tag in result.get("path", ""):
                return result
    except requests.RequestException as e:
        logging.error(f"Error fetching manifest info: {e}")
    return None

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
    
    logging.debug(f"EON_ID not found in build info: {build_info}")
    return "N/A"

def get_artifact_info(repo_name, artifact, headers):
    path_parts = artifact.get("path", "").split('/')
    if len(path_parts) < 2:
        logging.warning(f"Unexpected path structure for {artifact}")
        return None

    tag = path_parts[-1]
    image_name = '/'.join(path_parts[:-1])

    manifest_info = get_manifest_info(repo_name, image_name, tag, headers)
    if not manifest_info:
        logging.warning(f"No manifest found for {image_name}:{tag}")
        return None

    digest = manifest_info.get("actual_sha1", "")
    if not digest:
        logging.warning(f"No digest found for {image_name}:{tag}")
        return None

    eon_id = "N/A"
    build_info = get_build_info(f"{repo_name}/{image_name}", tag, headers)
    if build_info:
        eon_id = get_eon_id_from_build_info(build_info)

    formatted_digest = f"sha256:{digest}"

    return {
        "Resource_Name": f"{image_name}:{tag}",
        "CSP": "placeholder",
        "Resource_Type": "Container Image",
        "Unique_ID": formatted_digest,
        "EON_ID": eon_id,
        "Digest": formatted_digest,
        "Registry": f"{repo_name}/{image_name}"
    }

def process_repository(repo_name, headers):
    logging.info(f"Processing repository: {repo_name}")
    
    aql_query = f"""items.find(
        {{
            "repo": "{repo_name}",
            "$or": [
                {{"name": {{"$match": "*.tar.gz"}}}},
                {{"name": "manifest.json"}}
            ]
        }}
    ).include("repo", "path", "name")"""
    
    try:
        response = requests.post(SEARCH_API, data=aql_query, headers=headers, verify=False)
        response.raise_for_status()
        artifacts = response.json().get("results", [])
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")

    repo_details = []
    for artifact in tqdm(artifacts, desc=f"Processing artifacts in {repo_name}"):
        result = get_artifact_info(repo_name, artifact, headers)
        if result:
            repo_details.append(result)

    logging.info(f"Processed {len(repo_details)} valid artifacts in {repo_name}")
    return repo_details

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

This script does the following:

1. It uses AQL (Artifactory Query Language) to search for Docker images in each repository.
2. For each image, it retrieves the manifest information to get the digest.
3. It then fetches the build information associated with the image, which should contain the EON_ID.
4. The EON_ID is extracted from the build information's environment variables.
5. All the collected information is then saved to a CSV file.

To use this script:

1. Save it as `jfrog_artifact_info.py`.
2. Install the required libraries: `pip install requests tqdm`.
3. Run the script:
   ```
   python jfrog_artifact_info.py --token YOUR_ARTIFACTORY_TOKEN --output /path/to/output/directory
   ```

Replace `YOUR_ARTIFACTORY_TOKEN` with your actual JFrog Artifactory API token and specify the desired output directory.

This script should now correctly retrieve the EON_ID for Docker images pushed using the process you described. If you encounter any issues or need further modifications, please let me know.

---
Answer from Perplexity: pplx.ai/share
