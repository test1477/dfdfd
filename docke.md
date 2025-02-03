To delete CSV files that are older than 7 days in the `reports/ev` directory, you can add a step that runs a command to find and remove those files. The `find` command in Linux can be used to delete files older than 7 days.

Here's how you can update your workflow to include this step:

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

      - name: Delete Old EV Reports (older than 7 days)
        run: |
          echo "Deleting CSV files older than 7 days in reports/ev..."
          find reports/ev/*.csv -type f -mtime +7 -exec rm {} \; || echo "No files older than 7 days to delete"

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

### Key change:
- **Delete Old EV Reports**: Added a step to find and delete CSV files older than 7 days in the `reports/ev` directory using the `find` command:

  ```bash
  find reports/ev/*.csv -type f -mtime +7 -exec rm {} \;
  ```

This command will search for `.csv` files in the `reports/ev` directory, and if they are older than 7 days (`-mtime +7`), it will delete them (`-exec rm {} \;`).

If no files are older than 7 days, it will print `"No files older than 7 days to delete"`.
