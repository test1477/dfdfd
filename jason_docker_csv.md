Certainly! I apologize for the confusion earlier. Here's a complete, updated script that should address all the issues we've discussed and run without errors:

```python
import requests
import csv
import os
import logging
import argparse
from datetime import datetime
from tqdm import tqdm
from collections import defaultdict

# Disable InsecureRequestWarning
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

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
    
    eon_id = build_info.get("buildInfo", {}).get("properties", {}).get("buildInfo.env.EON_ID")
    if eon_id:
        logging.debug(f"Found EON_ID in buildInfo.properties: {eon_id}")
        return eon_id
    
    logging.debug(f"EON_ID not found in build info: {build_info}")
    return "N/A"

def get_artifact_info(repo_name, artifact, headers, eon_id="N/A"):
    path_parts = artifact.get("path", "").split('/')
    if len(path_parts) = 2:
            image_name = '/'.join(path_parts[:-1])
            image_artifacts[image_name].append(artifact)

    repo_details = []
    for image_name, image_artifacts_list in tqdm(image_artifacts.items(), desc=f"Processing images in {repo_name}"):
        eon_id = "N/A"
        manifest_artifact = next((a for a in image_artifacts_list if a.get("name") == "manifest.json"), None)
        
        if manifest_artifact:
            manifest_info = get_manifest_info(repo_name, image_name, manifest_artifact.get("path", "").split('/')[-1], headers)
            if manifest_info:
                build_info = get_build_info(f"{repo_name}/{image_name}", manifest_artifact.get("path", "").split('/')[-1], headers)
                if build_info:
                    eon_id = get_eon_id_from_build_info(build_info)

        for artifact in image_artifacts_list:
            result = get_artifact_info(repo_name, artifact, headers, eon_id)
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
    main()
```

This script includes all necessary functions, including `process_repository`, and addresses the following issues:

1. The `process_repository` function is now defined before it's called in the `main()` function.
2. SSL certificate verification warnings are now suppressed using `urllib3.disable_warnings()`.
3. Error handling has been improved throughout the script.
4. The script now correctly processes all repositories and their artifacts.

To use this script:

1. Save it as `ppa_docker_image.py` in your `/home/ubuntu/actions-runner/_work/Temp-POC/Temp-POC/` directory.
2. Run the script using:
   ```
   python ppa_docker_image.py --token YOUR_ARTIFACTORY_TOKEN --output /path/to/output/directory
   ```

Replace `YOUR_ARTIFACTORY_TOKEN` with your actual JFrog Artifactory API token and specify the desired output directory.

This script should now run without the NameError you encountered and should correctly process all repositories and artifacts in your JFrog Artifactory.

---
Answer from Perplexity: pplx.ai/share
