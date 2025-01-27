Here's a Python script that fetches artifacts for **all repositories** in a JFrog Artifactory instance using the JFrog REST API. The script retrieves repository names and iterates through them to fetch artifact metadata for each repository.

---

### Complete Script: Fetch Artifacts from All Repositories in JFrog Artifactory

```python
import os
import requests
import logging
import csv
from datetime import datetime


class ArtifactoryClient:
    def __init__(self, artifactory_url, token):
        """
        Initialize the Artifactory client.

        Args:
            artifactory_url (str): The base URL of the Artifactory instance.
            token (str): The authentication token for accessing Artifactory.
        """
        self.artifactory_url = artifactory_url.rstrip("/")
        self.token = token
        self.headers = {"Authorization": f"Bearer {self.token}"}

    def get_repositories(self):
        """
        Fetch all repositories from the Artifactory instance.

        Returns:
            list[str]: A list of repository names.
        """
        url = f"{self.artifactory_url}/api/repositories"
        response = requests.get(url, headers=self.headers)

        if response.status_code != 200:
            logging.error(f"Failed to fetch repositories: {response.status_code} - {response.text}")
            return []

        repos = [repo["key"] for repo in response.json()]
        logging.info(f"Fetched {len(repos)} repositories.")
        return repos

    def get_artifacts(self, repo_name):
        """
        Fetch artifacts for a specific repository.

        Args:
            repo_name (str): The name of the repository in Artifactory.

        Returns:
            list[dict]: A list of artifacts with their details.
        """
        url = f"{self.artifactory_url}/api/storage/{repo_name}"
        response = requests.get(url, headers=self.headers)

        if response.status_code != 200:
            logging.error(f"Failed to fetch artifacts for repo '{repo_name}': {response.status_code} - {response.text}")
            return []

        artifacts = self._extract_artifacts(response.json())
        return artifacts

    def _extract_artifacts(self, response_json):
        """
        Extract artifact details from the API response.

        Args:
            response_json (dict): JSON response from Artifactory API.

        Returns:
            list[dict]: Extracted artifact details.
        """
        artifacts = []
        if "children" in response_json:
            for child in response_json["children"]:
                if not child["uri"].endswith("/"):  # Ignore directories
                    artifact_url = f"{response_json['uri']}{child['uri']}"
                    artifacts.append({
                        "url": artifact_url,
                        "path": child["uri"],
                        "name": os.path.basename(child["uri"]),
                        "created": datetime.now().isoformat(),  # Replace with actual creation date if available
                    })
        return artifacts


class CSVReporter:
    def __init__(self, filename, headers):
        """
        Initialize a CSV reporter to write artifact data.

        Args:
            filename (str): The CSV file path.
            headers (list): A list of column headers for the CSV file.
        """
        self.filename = filename
        self.headers = headers

        # Ensure the directory exists
        os.makedirs(os.path.dirname(filename), exist_ok=True)

        # Open the file for writing
        self.file = open(filename, mode="w", newline="", encoding="utf-8")
        self.writer = csv.DictWriter(self.file, fieldnames=headers)
        self.writer.writeheader()

    def write_row(self, row):
        """Write a single row to the CSV file."""
        self.writer.writerow(row)

    def close(self):
        """Close the CSV file."""
        self.file.close()


if __name__ == "__main__":
    # Configure logging
    logging.basicConfig(level=logging.INFO)

    # Fetch environment variables for Artifactory URL and Token
    ARTIFACTORY_URL = os.environ.get("ARTIFACTORY_URL", "https://example.jfrog.io/artifactory")
    ARTIFACTORY_TOKEN = os.environ.get("ARTIFACTORY_TOKEN")

    if not ARTIFACTORY_TOKEN:
        raise ValueError("ARTIFACTORY_TOKEN is not set. Please export it as an environment variable.")

    # Initialize Artifactory client
    client = ArtifactoryClient(ARTIFACTORY_URL, ARTIFACTORY_TOKEN)

    # Fetch all repositories
    repositories = client.get_repositories()

    # Initialize CSV Reporter
    output_file = "output/all_artifacts_report.csv"
    reporter = CSVReporter(
        output_file,
        headers=[
            "repo_name",
            "package_type",
            "package_name",
            "version",
            "url",
            "creation_date",
            "license",
            "secarch",
            "artifactory_instance",
        ],
    )

    # Fetch artifacts for all repositories and write to the CSV
    for repo in repositories:
        logging.info(f"Processing repository: {repo}")
        artifacts = client.get_artifacts(repo)

        for artifact in artifacts:
            # Extract package name and version from the artifact URL (if applicable)
            package_name, version = None, None
            match = re.match(r".*/(?P<package>.+)-(?P<version>[0-9\.]+)\.[a-zA-Z0-9]+$", artifact["url"])
            if match:
                package_name = match.group("package")
                version = match.group("version")

            # Write the artifact data to the CSV file
            reporter.write_row({
                "repo_name": repo,
                "package_type": "Generic",  # Adjust this if package type can be determined
                "package_name": package_name or artifact["name"],
                "version": version or "N/A",
                "url": artifact["url"],
                "creation_date": artifact["created"],
                "license": "",  # Placeholder
                "secarch": "",  # Placeholder
                "artifactory_instance": ARTIFACTORY_URL,
            })

    # Close the CSV file
    reporter.close()

    logging.info(f"Artifact report generated: {output_file}")
```

---

### How It Works:

1. **Fetch Repositories:**
   - Uses the JFrog API (`/api/repositories`) to fetch all repository names in the Artifactory instance.

2. **Fetch Artifacts:**
   - Iterates over each repository and fetches artifact details using the `/api/storage/{repo_name}` endpoint.

3. **Extract Package Details:**
   - Parses artifact URLs to extract `package_name` and `version` where applicable (e.g., `package-1.0.0.tar.gz`).

4. **CSV Report:**
   - Writes the results to a CSV file (`output/all_artifacts_report.csv`) with the specified columns.

---

### Setup:
1. Export environment variables for Artifactory URL and token:
   ```bash
   export ARTIFACTORY_URL="https://your-artifactory-instance.jfrog.io/artifactory"
   export ARTIFACTORY_TOKEN="your_api_token_here"
   ```

2. Run the script:
   ```bash
   python artifactory_fetch_all.py
   ```

---

### Sample Output (CSV):
Example CSV output for two repositories:

```csv
repo_name,package_type,package_name,version,url,creation_date,license,secarch,artifactory_instance
repo1,Generic,package1,1.0.0,https://example.jfrog.io/artifactory/repo1/path/to/package1-1.0.0.tar.gz,2025-01-27T12:00:00,,,
repo2,Generic,package2,2.0.0,https://example.jfrog.io/artifactory/repo2/path/to/package2-2.0.0.zip,2025-01-27T12:00:00,,,
```

---

### Key Features:
1. **Handles All Repositories:** Automatically retrieves all repositories.
2. **Artifact Filtering:** Only includes non-directory artifacts in the report.
3. **CSV Report:** Writes all artifact details to a CSV file.
4. **Error Handling:** Logs errors if API requests fail.
5. **Dynamic Configuration:** Reads `ARTIFACTORY_URL` and `ARTIFACTORY_TOKEN` from environment variables for security.

Let me know if you'd like further customization!
