The error `CERTIFICATE_VERIFY_FAILED` occurs when Python's `requests` library is unable to verify the SSL certificate of the JFrog Artifactory server. This can happen if:

1. The server uses a self-signed certificate.
2. The server's certificate is not recognized by the CA (Certificate Authority) bundle used by Python.
3. The server's SSL configuration is misconfigured.

To resolve this issue, you have a few options:

---

### **Option 1: Disable SSL Verification (Not Recommended for Production)**
You can disable SSL verification by passing `verify=False` to the `requests.get()` calls. However, this is **not recommended for production environments** as it makes your script vulnerable to man-in-the-middle attacks.

Update the `requests.get()` calls in the script as follows:

```python
response = requests.get(url, headers=headers, verify=False)
```

Example:
```python
def fetch_packages(repo_key):
    """Fetch packages from a specific repository."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}'
    response = requests.get(url, headers=headers, verify=False)  # Disable SSL verification
    if response.status_code == 200:
        return response.json().get('children', [])
    else:
        print(f"Failed to fetch packages from {repo_key}: {response.status_code}")
        return []
```

---

### **Option 2: Use a Custom CA Bundle**
If your JFrog Artifactory server uses a self-signed certificate or a certificate issued by a private CA, you can provide the CA certificate to the `requests` library.

1. **Download the CA Certificate**:
   - Obtain the CA certificate (e.g., `.crt` or `.pem` file) for your Artifactory server.
   - Save it to a file, e.g., `artifactory-ca.crt`.

2. **Update the Script**:
   - Pass the path to the CA certificate file using the `verify` parameter in `requests.get()`.

Example:
```python
def fetch_packages(repo_key):
    """Fetch packages from a specific repository."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}'
    response = requests.get(url, headers=headers, verify='/path/to/artifactory-ca.crt')  # Use custom CA bundle
    if response.status_code == 200:
        return response.json().get('children', [])
    else:
        print(f"Failed to fetch packages from {repo_key}: {response.status_code}")
        return []
```

---

### **Option 3: Add the Certificate to Python's CA Bundle**
If you have the CA certificate, you can add it to Python's CA bundle. This ensures that all `requests` calls in your script (and other Python applications) trust the certificate.

1. **Locate Python's CA Bundle**:
   - Run the following command to find the location of Python's CA bundle:
     ```python
     import certifi
     print(certifi.where())
     ```
   - This will output the path to the CA bundle file (e.g., `lib/python3.x/site-packages/certifi/cacert.pem`).

2. **Add the Certificate**:
   - Append the contents of your CA certificate (e.g., `artifactory-ca.crt`) to the CA bundle file.

3. **Restart Your Script**:
   - No changes to the script are required. Python will now trust the certificate.

---

### **Option 4: Use Environment Variables (for Testing)**
If you're running the script in a test environment and want to bypass SSL verification globally, you can set the `PYTHONHTTPSVERIFY` environment variable to `0`.

On Linux/Mac:
```bash
export PYTHONHTTPSVERIFY=0
python report_script.py
```

On Windows:
```cmd
set PYTHONHTTPSVERIFY=0
python report_script.py
```

This disables SSL verification for all `requests` calls in the script.

---

### **Option 5: Update System Certificates**
If the server's certificate is issued by a recognized CA but Python still cannot verify it, your system's CA certificates might be outdated. Update them as follows:

- **On Linux**:
  ```bash
  sudo apt-get update
  sudo apt-get install ca-certificates
  sudo update-ca-certificates
  ```

- **On Mac**:
  ```bash
  brew update
  brew install ca-certificates
  ```

- **On Windows**:
  Update your system's root certificates through Windows Update.

---

### Updated Script with SSL Verification Disabled (for Testing)
Here’s the updated script with SSL verification disabled (for testing purposes only):

```python
import requests
import csv
from datetime import datetime

# Configuration
ARTIFACTORY_URL = 'https://your-artifactory-instance/artifactory'  # Replace with your Artifactory URL
API_KEY = 'your-api-key'  # Replace with your API key or use username/password
CSV_FILE = 'all_packages_report.csv'

# Headers for authentication
headers = {
    'X-JFrog-Art-Api': API_KEY
}

def fetch_repositories():
    """Fetch all repositories from Artifactory."""
    repos_url = f'{ARTIFACTORY_URL}/api/repositories'
    response = requests.get(repos_url, headers=headers, verify=False)  # Disable SSL verification
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

def fetch_packages(repo_key):
    """Fetch packages from a specific repository."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}'
    response = requests.get(url, headers=headers, verify=False)  # Disable SSL verification
    if response.status_code == 200:
        return response.json().get('children', [])
    else:
        print(f"Failed to fetch packages from {repo_key}: {response.status_code}")
        return []

def fetch_package_metadata(repo_key, package_path):
    """Fetch metadata for a specific package."""
    url = f'{ARTIFACTORY_URL}/api/storage/{repo_key}/{package_path}'
    response = requests.get(url, headers=headers, verify=False)  # Disable SSL verification
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch metadata for {package_path}: {response.status_code}")
        return None

def extract_package_info(repo_key, package_path, repo_type):
    """Extract package information based on repository type."""
    metadata = fetch_package_metadata(repo_key, package_path)
    if not metadata:
        return None

    package_name = metadata.get('name', 'N/A')
    version = metadata.get('version', 'N/A')
    url = metadata.get('downloadUri', 'N/A')
    created_date = metadata.get('created', 'N/A')

    # Convert timestamp to readable date
    if created_date != 'N/A':
        created_date = datetime.fromtimestamp(created_date / 1000).strftime('%Y-%m-%d %H:%M:%S')

    return [repo_key, repo_type, package_name, version, url, created_date]

def save_to_csv(data, filename):
    """Save the report data to a CSV file."""
    with open(filename, mode='w', newline='') as file:
        writer = csv.writer(file)
        writer.writerow(['Repo Name', 'Package Type', 'Package Name', 'Version', 'URL', 'Created Date'])
        for row in data:
            writer.writerow(row)

def main():
    # Fetch all repositories
    repositories = fetch_repositories()
    if not repositories:
        return

    report_data = []

    for repo in repositories:
        repo_key = repo['key']
        repo_type = repo['type']
        print(f"Processing repository: {repo_key} (Type: {repo_type})")

        packages = fetch_packages(repo_key)
        for pkg in packages:
            package_path = pkg['uri'][1:]  # Remove leading slash
            package_info = extract_package_info(repo_key, package_path, repo_type)

            if package_info:
                report_data.append(package_info)

    # Save the report to CSV
    save_to_csv(report_data, CSV_FILE)
    print(f"Report saved to {CSV_FILE}")

if __name__ == '__main__':
    main()
```

---

### Recommendation:
- For **production environments**, use **Option 2 (Custom CA Bundle)** or **Option 3 (Update Python's CA Bundle)**.
- For **testing purposes**, you can use **Option 1 (Disable SSL Verification)** or **Option 4 (Environment Variable)**.

Let me know if you need further assistance!
