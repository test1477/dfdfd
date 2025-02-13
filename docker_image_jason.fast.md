Certainly! I'll modify the script to accept command-line arguments for the token and output directory. Here's the updated version that works with the format you specified:

```python
import requests
import csv
import os
import logging
import argparse
from datetime import datetime
from tqdm import tqdm

# Logging setup
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# Constants (Replace with your JFrog URL)
JFROG_URL = "https://your-jfrog-instance/artifactory"
ARTIFACTORY_API = f"{JFROG_URL}/api/storage"
REPOSITORIES_API = f"{JFROG_URL}/api/repositories"
BUILDS_API = f"{JFROG_URL}/api/build"
MAX_RETRIES = 3

def parse_arguments():
    parser = argparse.ArgumentParser(description="Fetch Docker image details from JFrog Artifactory.")
    parser.add_argument("--token", required=True, help="Artifactory API token")
    parser.add_argument("--output", required=True, help="Output directory for the CSV report")
    return parser.parse_args()

def get_headers(token):
    return {
        "Authorization": f"Bearer {token}",
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

def list_artifacts(repo_name, headers):
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1&listFolders=0"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

def get_build_info(build_name, build_number, headers):
    url = f"{BUILDS_API}/{build_name}/{build_number}"
    try:
        response = requests.get(url, headers=headers, verify=False)
        if response.status_code == 200:
            return response.json()
        else:
            logging.warning(f"Failed to fetch build info for {build_name}/{build_number}: {response.status_code}")
    except requests.RequestException as e:
        logging.warning(f"Error fetching build info: {e}")
    return None

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

            properties = data.get("properties", {})
            build_name = properties.get("build.name")
            build_number = properties.get("build.number")
            
            eon_id = "N/A"
            if build_name and build_number:
                build_info = get_build_info(build_name, build_number, headers)
                if build_info:
                    eon_id = build_info.get("buildInfo", {}).get("properties", {}).get("buildInfo.env.EON_ID", "N/A")

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

To use this script:

1. Save it as `script_name.py` (or any name you prefer).
2. Make sure you have the required libraries installed (`requests`, `tqdm`).
3. Run the script using:

   ```
   python script_name.py --token YOUR_ARTIFACTORY_TOKEN --output /path/to/output/directory
   ```

   Replace `YOUR_ARTIFACTORY_TOKEN` with your actual JFrog Artifactory API token and `/path/to/output/directory` with the directory where you want the CSV file to be saved.

4. The script will process all Docker repositories in your JFrog Artifactory and create a CSV file named `EV_EOL_YYYY-MM-DD.csv` in the specified output directory.

Remember to replace the `JFROG_URL` constant at the beginning of the script with your actual JFrog Artifactory URL.

---
Answer from Perplexity: pplx.ai/share
