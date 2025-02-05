The message describes a new **GitHub Action integration** that enforces compliance checks for **TRP (Technical Risk Profile)** and **RDA (likely Release Deployment Approval)** before allowing code merges to the default branch. Here's what this means for your team:

---

### **Key Components**
1. **SDLC Gate Control API**  
   The system uses an API endpoint (`http://sdic-hygiene-prod-service.ms.com/rest/api/1.0/sdic-gate-control`) to validate whether a release meets TRP requirements. 
   - Accepts `control_gates` (e.g., `trp`) and an `eonid` (entity identifier).
   - Returns whether the merge should be blocked (`block_release`), compliance status (`in_scope`), and violation details.

2. **Three Response Scenarios**
   - **Scenario 0 (Not in Scope)**: No TRP required → merge allowed.
   - **Scenario 1 (Soft Block)**: TRP missing but merge allowed until **15 Feb 2025** → warning issued.
   - **Scenario 2 (Hard Block)**: TRP missing → merge blocked (enforced post-15 Feb 2025).

3. **GitHub Action Workflow**  
   The action will:
   - Trigger on pull requests targeting the default branch.
   - Call the SDLC Gate Control API to validate TRP/RDA compliance.
   - Pass/fail based on the API’s `block_release` flag.

---

### **Implementation Requirements**
#### **1. Branch Protection Rules**
Configure the default branch (e.g., `main`) to:
- Require the new GitHub Action check to pass before merging ([Search Result 1][1]).
- Enable "Require status checks to pass" and "Require branches to be up to date" in repository settings.

#### **2. GitHub Action Configuration**
Example workflow structure:
```yaml
name: TRP/RDA Compliance Check
on:
  pull_request:
    branches: [ main ]
    types: [ closed ]

jobs:
  check-compliance:
    runs-on: ubuntu-latest
    steps:
      - name: Validate TRP/RDA
        run: |
          # Call SDLC Gate Control API with eonid and control_gates
          RESPONSE=$(curl -X POST "API_ENDPOINT" -d '{"control_gates": "trp,rda", "eonid": "265741"}')
          BLOCK_RELEASE=$(echo $RESPONSE | jq '.block_release')

          if [ "$BLOCK_RELEASE" = "true" ]; then
            echo "TRP/RDA check failed. Merge blocked." && exit 1
          fi
```

#### **3. Post-15 Feb 2025 Enforcement**
- Cloud-related releases without approved TRPs will be blocked ([Scenario 2](#scenario-2-hard-block)).
- On-premises GRNs will follow a TBD timeline.

---

### **Action Items for Your Team**
1. **Immediate Steps** (Pre-15 Feb 2025):
   - Integrate the GitHub Action using the API.
   - Address soft-block warnings by creating TRPs in [Cutover](http://cutover/).
2. **Long-Term**:
   - Monitor API responses for `violation_message` guidance.
   - Prepare for hard-block enforcement by ensuring all cloud releases have TRPs.

This setup aligns with **SDLC governance** and GitHub’s security automation capabilities ([Search Result 17][17]). For debugging merge-trigger workflows, see community discussions on `pull_request: types: [closed]` conditions ([Search Result 3][3]).

Citations:
[1] https://calmcode.io/course/github-actions/prevent-merge
[2] https://github.blog/enterprise-software/github-actions-for-security-compliance/
[3] https://github.com/orgs/community/discussions/67160
[4] https://github.blog/security/supply-chain-security/slsa-3-compliance-with-github-actions/
[5] https://stackoverflow.com/questions/62846375/perform-github-action-when-trying-to-merge-branch
[6] https://docs.github.com/en/actions
[7] https://docs.github.com/en/actions/use-cases-and-examples/project-management/using-github-actions-for-project-management
[8] https://github.com/resources/articles/software-development/what-is-sdlc
[9] https://www.reddit.com/r/nextjs/comments/1cn2e9t/project_structure_setup_for_a_team_project_with/
[10] https://github.com/actions/checkout/issues/15
[11] https://www.youtube.com/watch?v=g3qfBeat-PM
[12] https://docs.github.com/en/actions/about-github-actions/understanding-github-actions
[13] https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue
[14] https://github.com/RDA-DMP-Common/user-stories/issues/109
[15] https://stackoverflow.com/questions/61255989/dynamically-retrieve-github-actions-secret
[16] https://github.com/orgs/community/discussions/53922
[17] https://www.adesso.co.hu/en/news/blog/enhancing-sdlc-automation-via-github-actions.jsp
[18] https://peterullrich.com/setup-recurring-jobs-with-github-actions
[19] https://github.com/dorny/paths-filter/issues/201
[20] https://www.legitsecurity.com/blog/github-actions-that-open-the-door-to-cicd-pipeline-attacks

---
Answer from Perplexity: pplx.ai/share
