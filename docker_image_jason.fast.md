Certainly! Here's the full script incorporating the changes we discussed, including the new `get_eon_id_from_xray` function and the updated `get_artifact_info` function. The script follows the previous format and includes the EV_EOL CSV output:

```python
import requests
import csv
import os
import concurrent.futures
import logging
import argparse
from datetime import datetime
from tqdm import tqdm

# Logging setup
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# Constants
JFROG_URL = "https://frigate.jfrog.io"
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"
MAX_WORKERS = 10
MAX_RETRIES = 3

def parse_arguments():
    """Parse command-line arguments."""
    parser = argparse.ArgumentParser(description="Fetch Docker image details from JFrog Artifactory.")
    parser.add_argument("--token", required=True, help="Artifactory API token")
    parser.add_argument("--output", required=True, help="Output directory for the CSV report")
    return parser.parse_args()

def get_headers(token):
    """Return headers for API authentication."""
    return {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json"
    }

def list_repositories(headers):
    """Retrieve the list of Docker repositories from Artifactory."""
    url = f"{JFROG_URL}/artifactory/api/repositories"
    response = requests.get(url, headers=headers, verify=False)
    response.raise_for_status()
    return [repo for repo in response.json() if repo.get("packageType") == "Docker"]

def list_artifacts(repo_name, headers):
    """List artifacts in a given repository."""
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1&listFolders=0"
    response = requests.get(url, headers=headers, verify=False)
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        logging.error(f"Error fetching artifacts for {repo_name}: {response.text}")
        return []

def get_eon_id_from_xray(repo_name, artifact_path, headers):
    """Fetch EON ID from JFrog Xray API."""
    xray_url = f"{JFROG_URL}/xray/api/v1/artifacts/{repo_name}/{artifact_path}"
    try:
        response = requests.get(xray_url, headers=headers, verify=False)
        if response.status_code == 200:
            xray_data = response.json()
            eon_id = xray_data.get("eon_id", "N/A")
            if eon_id == "N/A":
                logging.warning(f"EON_ID not found for {artifact_path} in Xray")
            return eon_id
        else:
            logging.warning(f"Failed to fetch EON_ID for {artifact_path} from Xray: {response.text}")
    except requests.RequestException as e:
        logging.warning(f"Failed to fetch EON_ID for {artifact_path} from Xray: {e}")
    return "N/A"

def get_artifact_info(repo_name, artifact, headers):
    """Retrieve artifact details, including EON ID if available."""
    artifact_path = artifact.get("uri", "").lstrip("/")
    if ".jfrog" in artifact_path:
        return None

    url = f"{ARTIFACTORY_API}/{repo_name}/{artifact_path}"
    for _ in range(MAX_RETRIES):
        try:
            response = requests.get(url, headers=headers, verify=False, timeout=10)
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

            # Fetch EON ID from Xray
            eon_id = get_eon_id_from_xray(repo_name, artifact_path, headers)

            # Ensure the digest is correctly formatted
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
        except IndexError as e:
            logging.error(f"IndexError processing {artifact_path}: {e}")
            return None

    logging.error(f"Failed to fetch {url} after {MAX_RETRIES} attempts")
    return None

def process_repository(repo_name, headers):
    """Process a repository and retrieve artifact details."""
    logging.info(f"Processing repository: {repo_name}")
    artifacts = list_artifacts(repo_name, headers)
    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")

    repo_details = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        future_to_artifact = {executor.submit(get_artifact_info, repo_name, artifact, headers): artifact for artifact in artifacts}
        for future in tqdm(concurrent.futures.as_completed(future_to_artifact), total=len(artifacts), desc=f"Processing {repo_name}"):
            try:
                result = future.result()
                if result:
                    repo_details.append(result)
            except Exception as e:
                logging.error(f"Error processing artifact in {repo_name}: {e}")

    logging.info(f"Processed {len(repo_details)} valid artifacts in {repo_name}")
    return repo_details

def save_to_csv(data, output_dir):
    """Save data to a CSV file with the correct header order."""
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = os.path.join(output_dir, f"EV_EOL_{current_date}.csv")

    headers = ['Resource_Name', 'CSP', 'Resource_Type', 'Unique_ID', 'EON_ID', 'Digest', 'Registry']

    os.makedirs(os.path.dirname(output_file), exist_ok=True)
    
    with open(output_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    logging.info(f"Data has been written to {output_file}")

def main():
    """Main execution function."""
    args = parse_arguments()
    headers = get_headers(args.token)

    repositories = list_repositories(headers)
    logging.info(f"Found {len(repositories)} Docker repositories")

    all_data = []
    for repo in repositories:
        repo_data = process_repository(repo["key"], headers)
        all_data.extend(repo_data)

    save_to_csv(all_data, args.output)
    logging.info("Script execution completed")

if __name__ == "__main__":
    requests.packages.urllib3.disable_warnings()
    main()
```

This script now includes:

1. The new `get_eon_id_from_xray` function to fetch EON_ID from JFrog Xray.
2. An updated `get_artifact_info` function that uses the Xray API to fetch EON_ID.
3. The same CSV output format as before, with the filename starting with "EV_EOL_".

Make sure to run this script with the appropriate arguments:

```
python script_name.py --token YOUR_ARTIFACTORY_TOKEN --output /path/to/output/directory
```

Remember to replace `YOUR_ARTIFACTORY_TOKEN` with your actual JFrog Artifactory API token and specify the desired output directory.

---
Answer from Perplexity: pplx.ai/share
