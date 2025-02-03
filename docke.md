To delete CSV files that are older than 7 days in the JFrog Artifactory `/reports/ev` repository, you can use the JFrog CLI or `curl` to first list the files, check their dates, and then delete the older ones. JFrog Artifactory does not directly support the `find` command like a local filesystem, so you'd have to interact with the Artifactory REST API to get the list of files and their metadata.

Here’s how you can update the workflow to delete files older than 7 days from the `/reports/ev` repository on JFrog:

### Updated workflow with file deletion in JFrog Artifactory:

```yaml
name: Generate JFrog Reports

on:
  schedule:
    - cron: "0 9 * * *"  # Runs daily at 9 AM UTC
  workflow_dispatch:  # Allows manual triggering

jobs:
  generate-report:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set Up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.9"

      - name: Install Dependencies
        run: pip install requests

      - name: Run Report Script for EV
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
        run: |
          echo "Generating report for EV..."
          python report_script.py --org ev --jfrog-url "https://frigate.io/artifactory" --output reports/ev || echo "EV report generation failed"
          ls -lh reports/ev || echo "EV directory is empty"

      - name: Run Report Script for PPA
        env:
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          echo "Generating report for PPA..."
          python report_script.py --org ppa --jfrog-url "https://ppa.jfrog.io/artifactory" --output reports/ppa || echo "PPA report generation failed"
          ls -lh reports/ppa || echo "PPA directory is empty"

      - name: Validate Reports Before Upload
        run: |
          ls -lh reports/ev || echo "EV report missing"
          ls -lh reports/ppa || echo "PPA report missing"

      - name: Delete Old EV Reports from JFrog (older than 7 days)
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
        run: |
          echo "Deleting CSV files older than 7 days in JFrog /reports/ev..."
          # List the files in the reports/ev repository and check their creation date
          files=$(curl -s -H "X-JFrog-Art-Api:${JFROG_API_KEY_EV}" "https://frigate.io/artifactory/api/storage/reports/ev?list&listFolders=0" | jq -r '.files[] | select(.uri | test(".csv$")) | .uri')
          for file in $files; do
            # Get the file details and check the last modified date
            last_modified=$(curl -s -H "X-JFrog-Art-Api:${JFROG_API_KEY_EV}" "https://frigate.io/artifactory/api/storage/reports/ev$file" | jq -r '.lastModified')
            last_modified_timestamp=$(date -d "$last_modified" +%s)
            current_timestamp=$(date +%s)
            diff=$(( (current_timestamp - last_modified_timestamp) / 86400 ))  # Difference in days
            if [ $diff -gt 7 ]; then
              # Delete the file if it is older than 7 days
              echo "Deleting $file (older than 7 days)"
              curl -X DELETE -H "X-JFrog-Art-Api:${JFROG_API_KEY_EV}" "https://frigate.io/artifactory/reports/ev$file" || echo "Failed to delete $file"
            fi
          done

      - name: Upload Reports to JFrog (EV)
        env:
          JFROG_API_KEY_EV: ${{ secrets.JFROG_API_KEY_EV }}
        run: |
          if [ -f "reports/ev/EV_EOL_$(date +%m-%d-%Y).csv" ]; then
            echo "Uploading EV report..."
            curl -H "X-JFrog-Art-Api:${JFROG_API_KEY_EV}" -T reports/ev/EV_EOL_$(date +%m-%d-%Y).csv "https://frigate.io/artifactory/reports/ev/EV_EOL_$(date +%m-%d-%Y).csv" || echo "EV upload failed"
          else
            echo "EV report file not found, skipping upload."
          fi

      - name: Upload Reports to JFrog (PPA)
        env:
          JFROG_API_KEY_PPA: ${{ secrets.JFROG_API_KEY_PPA }}
        run: |
          if [ -f "reports/ppa/EV_EOL_$(date +%m-%d-%Y).csv" ]; then
            echo "Uploading PPA report..."
            curl -H "X-JFrog-Art-Api:${JFROG_API_KEY_PPA}" -T reports/ppa/EV_EOL_$(date +%m-%d-%Y).csv "https://ppa.jfrog.io/artifactory/reports/ppa/EV_EOL_$(date +%m-%d-%Y).csv" || echo "PPA upload failed"
          else
            echo "PPA report file not found, skipping upload."
          fi
```

### Key changes:
1. **Deleting old files from JFrog**:
   - The workflow fetches the list of CSV files in the `/reports/ev` repository using the JFrog Artifactory Storage API.
   - It checks the `lastModified` date for each file, calculates the difference from the current date, and deletes files that are older than 7 days.
   - Uses `jq` to parse JSON responses and `curl` to delete the files.

### Notes:
- Make sure you have the `jq` tool installed on the runner since it's used to parse the JSON response from Artifactory.
- The script uses the `X-JFrog-Art-Api` header to authenticate requests, and the `DELETE` method is used to remove old files.

This workflow will now delete CSV files in `/reports/ev` older than 7 days from the JFrog repository before uploading the new reports.
