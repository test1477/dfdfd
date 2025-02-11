Here's your updated script with the same format you requested, ensuring the output file includes the current date dynamically. The script now also accepts command-line arguments for flexibility in GitHub Actions.  

### **Updates Applied:**
✅ Output file now follows the format: `EV_EOL_<current_date>.csv`  
✅ Uses `argparse` to allow setting the output directory dynamically  
✅ Keeps all existing functionalities intact  

---

### **Updated Script:**
```python
import requests
import csv
import os
import concurrent.futures
import logging
import argparse
from datetime import datetime
from tqdm import tqdm

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

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

def get_artifact_info(repo_name, artifact, headers):
    """Retrieve detailed information about an artifact."""
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
    """Save data to a CSV file with the current date in the filename."""
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = os.path.join(output_dir, f"EV_EOL_{current_date}.csv")

    headers = ['Docker Image Name', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path', 'CSP']
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

---

### **Usage in GitHub Actions**
Modify your GitHub Actions YAML file like this:

```yaml
jobs:
  fetch_jfrog_data:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: pip install requests tqdm

      - name: Run JFrog Report Script
        run: |
          python script.py --token "${{ secrets.ARTIFACTORY_TOKEN }}" --output "./reports"
```

---

### **What’s Improved?**
✔ **Flexible Output Path:** Can be set dynamically with `--output`  
✔ **Date in Filename:** `EV_EOL_YYYY-MM-DD.csv` format  
✔ **Secure Token Handling:** Passed as an argument instead of hardcoding  
✔ **GitHub Actions Ready:** Can be used in a CI/CD pipeline  

This should now be ready for **GitHub Actions** and **local execution**. 🚀
