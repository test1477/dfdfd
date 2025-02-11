I've reformatted your script to align with the **GitHub Actions-friendly format**, following the same structure as before while improving maintainability and flexibility.  

### **🔧 Key Updates:**
✅ **Uses `argparse` for input parameters** (e.g., org name, JFrog URL, output path).  
✅ **Reads `ARTIFACTORY_TOKEN` from environment variables** (to avoid hardcoding secrets).  
✅ **Ensures compatibility with GitHub Actions by dynamically handling inputs and outputs.**  
✅ **Logs progress with `tqdm` for better visibility in CI/CD logs.**  

---

### **🚀 Updated GitHub Actions-Friendly Script**
```python
import os
import csv
import requests
import logging
import argparse
import concurrent.futures
from tqdm import tqdm

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# Constants
MAX_WORKERS = 10
MAX_RETRIES = 3

# Parse command-line arguments
parser = argparse.ArgumentParser(description="Generate JFrog Docker reports.")
parser.add_argument('--org', required=True, help="The organization name (e.g., 'ev' or 'ppa').")
parser.add_argument('--jfrog-url', required=True, help="The JFrog Artifactory base URL.")
parser.add_argument('--output', required=True, help="The output directory for the generated report.")
args = parser.parse_args()

# JFrog API details
JFROG_URL = args.jfrog_url.rstrip("/")  # Ensure no trailing slash
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"

# Fetch Artifactory token from environment variables
ARTIFACTORY_TOKEN = os.getenv(f'JFROG_API_KEY_{args.org.upper()}')
if not ARTIFACTORY_TOKEN:
    logging.error(f"Error: JFROG_API_KEY_{args.org.upper()} is not set in environment variables.")
    exit(1)

# Headers for API authentication
HEADERS = {
    "Authorization": f"Bearer {ARTIFACTORY_TOKEN}",
    "Accept": "application/json"
}

# Function to list all repositories
def list_repositories():
    url = f"{JFROG_URL}/artifactory/api/repositories"
    try:
        response = requests.get(url, headers=HEADERS, verify=False)
        response.raise_for_status()
        return [repo for repo in response.json() if repo.get("packageType") == "Docker"]
    except requests.RequestException as e:
        logging.error(f"Failed to fetch repositories: {e}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1&listFolders=0"
    try:
        response = requests.get(url, headers=HEADERS, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

# Function to fetch artifact details
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
                "CSP": "frigate.jfrog.io"
            }
        except requests.RequestException as e:
            logging.warning(f"Error fetching {url}: {e}. Retrying...")

    logging.error(f"Failed to fetch {url} after {MAX_RETRIES} attempts")
    return None

# Function to process a repository
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

# Function to save data to CSV
def save_to_csv(data, filename):
    headers = ['Docker Image Name', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path', 'CSP']
    os.makedirs(os.path.dirname(filename), exist_ok=True)

    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)

    logging.info(f"Data has been written to {filename}")

# Main function
def main():
    repositories = list_repositories()
    logging.info(f"Found {len(repositories)} Docker repositories")

    all_data = []
    for repo in repositories:
        repo_data = process_repository(repo["key"])
        all_data.extend(repo_data)

    # Save to CSV with a dynamic file name
    output_file = os.path.join(args.output, f"JFrog_Docker_Report.csv")
    save_to_csv(all_data, output_file)

if __name__ == "__main__":
    main()
```

---

### **💡 Key GitHub Actions-Friendly Changes**
✅ **Uses `argparse` for input arguments**  
- `--org`: Organization name (used to fetch API key)  
- `--jfrog-url`: Base JFrog URL (avoids hardcoding)  
- `--output`: Output directory (can be dynamically set in workflows)  

✅ **Environment Variable for API Key**  
- Reads from `JFROG_API_KEY_{ORG}` instead of hardcoding.  
- Ensures security in CI/CD pipelines.  

✅ **Ensures Logs Are GitHub Actions-Friendly**  
- Uses `logging.info()` instead of `print()`.  
- `tqdm` progress bars work well in GitHub logs.  

✅ **Dynamic File Naming for Outputs**  
- The output file is named **`JFrog_Docker_Report.csv`**, and the output path is set via `--output`, making it CI/CD compatible.  

---

### **🚀 How to Use in GitHub Actions**
```yaml
- name: Run JFrog Report Script
  run: python jfrog_report.py --org "ev" --jfrog-url "https://frigate.jfrog.io" --output "${{ github.workspace }}/output"
  env:
    JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
```

---

### **📌 Summary**
✅ **Now fully compatible with GitHub Actions**  
✅ **Uses environment variables for secrets**  
✅ **Provides detailed logs and progress tracking**  
✅ **Accepts dynamic inputs via `argparse`**  

This should now **work perfectly in your CI/CD pipeline**! 🚀 Let me know if you need any changes! 😊
