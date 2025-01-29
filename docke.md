Here's an updated `report_script.py` that fetches package metadata from JFrog Artifactory, filters for Docker packages, and saves the report in CSV format.  

### **`report_script.py`**
```python
import os
import requests
import argparse
import csv
from datetime import datetime

def get_artifactory_packages(jfrog_url, api_key, repo_name):
    headers = {"X-JFrog-Art-Api": api_key}
    url = f"{jfrog_url}/api/search/aql"
    query = f"""
        items.find({{"repo": "{repo_name}"}}).include("repo", "path", "name", "created", "size", "property")
    """
    
    response = requests.post(url, headers=headers, data=query)
    if response.status_code == 200:
        return response.json().get('results', [])
    else:
        print(f"Error fetching data: {response.text}")
        return []

def process_packages(packages, org, jfrog_url):
    data = []
    for package in packages:
        if not package.get('repo').startswith("docker"):  # Filter only Docker packages
            continue

        package_name = package.get('path', '')  # Use full repository path
        version = package.get('name', '').split("/")[-1]  # Extract version
        created_date = package.get('created', '')
        repo_name = package.get('repo', '')
        artifactory_instance = jfrog_url.split("//")[-1]

        data.append([repo_name, package_name, version, created_date, artifactory_instance])

    return data

def save_report(data, output_path, org):
    if not os.path.exists(output_path):
        os.makedirs(output_path)

    filename = f"{output_path}/EV_EOL_{datetime.today().strftime('%Y-%m-%d')}.csv"
    with open(filename, mode="w", newline="") as file:
        writer = csv.writer(file)
        writer.writerow(["repo_name", "package_name", "version", "created_date", "artifactory_instance"])
        writer.writerows(data)

    print(f"Report saved: {filename}")

def main():
    parser = argparse.ArgumentParser(description="Fetch and generate JFrog Artifactory package reports")
    parser.add_argument("--org", required=True, help="Organization name (ev or ppa)")
    parser.add_argument("--jfrog-url", required=True, help="JFrog Artifactory URL")
    parser.add_argument("--output", required=True, help="Output folder for reports")
    args = parser.parse_args()

    api_key = os.getenv("JFROG_API_KEY")
    if not api_key:
        print("Error: JFROG_API_KEY is not set")
        return

    repo_name = "docker-repo"  # Modify if needed
    packages = get_artifactory_packages(args.jfrog_url, api_key, repo_name)
    processed_data = process_packages(packages, args.org, args.jfrog_url)
    save_report(processed_data, args.output, args.org)

if __name__ == "__main__":
    main()
```

---

### **How It Works**
1. **Fetches package metadata** from JFrog Artifactory using AQL queries.
2. **Filters only Docker packages** (`repo` starts with `"docker"`).
3. **Extracts**:
   - `repo_name`
   - `package_name` (repository path)
   - `version`
   - `created_date`
   - `artifactory_instance`
4. **Saves the report** as `EV_EOL_<current_date>.csv`.

---

### **Usage**
The script is triggered by **GitHub Actions**, but you can also run it manually:
```sh
export JFROG_API_KEY="your_api_key"
python report_script.py --org ev --jfrog-url "https://frigate.io/artifactory" --output reports/ev
```

Let me know if you need any tweaks! 🚀


Got it! If JFrog API keys or tokens are required instead of username/password for authentication, we can modify the script to use them.

---

### **Updated GitHub Actions Workflow:**
```yaml
name: JFrog Report Automation

on:
  schedule:
    - cron: '0 9 * * *'  # Runs daily at 9 AM UTC
  workflow_dispatch:  # Allows manual trigger

jobs:
  generate-report:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'

      - name: Install Dependencies
        run: |
          pip install requests pandas  # Add any other dependencies if needed

      - name: Run Report Script for EV
        env:
          JFROG_API_KEY: ${{ secrets.JFROG_API_KEY_EV }}
          JFROG_URL: "https://frigate.io/artifactory"
        run: |
          python report_script.py --org ev --jfrog-url $JFROG_URL --output reports/ev

      - name: Run Report Script for PPA
        env:
          JFROG_API_KEY: ${{ secrets.JFROG_API_KEY_PPA }}
          JFROG_URL: "https://ppa.jfrog.io/artifactory"
        run: |
          python report_script.py --org ppa --jfrog-url $JFROG_URL --output reports/ppa

      - name: Upload Reports to Frigate JFrog (EV)
        env:
          JFROG_API_KEY: ${{ secrets.JFROG_API_KEY_EV }}
          JFROG_URL: "https://frigate.io/artifactory"
        run: |
          for file in reports/ev/*; do
            curl -H "X-JFrog-Art-Api:$JFROG_API_KEY" -T "$file" "$JFROG_URL/reports/ev/$(basename $file)"
          done

      - name: Upload Reports to Frigate JFrog (PPA)
        env:
          JFROG_API_KEY: ${{ secrets.JFROG_API_KEY_PPA }}
          JFROG_URL: "https://ppa.jfrog.io/artifactory"
        run: |
          for file in reports/ppa/*; do
            curl -H "X-JFrog-Art-Api:$JFROG_API_KEY" -T "$file" "$JFROG_URL/reports/ppa/$(basename $file)"
          done
```

---

### **Key Changes**
✅ **Uses JFrog API keys for authentication**  
✅ **Uploads each report file using `curl` with `X-JFrog-Art-Api` header**  
✅ **Ensures correct paths for `reports/ev/` and `reports/ppa/` uploads**  

---

### **Next Steps**
- Ensure the secrets `JFROG_API_KEY_EV` and `JFROG_API_KEY_PPA` are stored in **GitHub Actions Secrets**.
- Modify `report_script.py` to dynamically handle JFrog API calls if needed.

Let me know if you need more changes! 🚀
