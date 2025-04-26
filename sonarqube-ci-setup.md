# Documentation: Journey to Set Up SonarCloud + Discord Notification in GitHub Actions

## Goal
We wanted to create a GitHub Actions workflow that:
- Runs a SonarCloud (or SonarQube CE) analysis after Java build.
- Waits for the analysis to complete.
- Sends a Discord notification if the analysis is ready.

## Attempt 1: Direct Discord Notification with Full Metrics

### What we tried:
- After Sonar scan, use Bash to `curl` SonarCloud APIs to:
  - Check if analysis completed (`status == SUCCESS`)
  - Fetch metrics: Coverage, Bugs, Smells, Duplications, Security Hotspots, etc.
- Build a JSON Discord payload using `cat <<EOF`.

### Problems faced:
- GitHub Actions YAML parser crashed when trying to handle `cat <<EOF` inside `run: |`.
- EOF must be at column 0 in Bash, but YAML demands indentation.
- Result: GitHub threw YAML syntax errors:
  ```
  error parsing called workflow: You have an error in your yaml syntax
  ```

## Attempt 2: Move to Inline JSON with `curl -d`

### What we tried:
- Build JSON inline in Bash (`-d "{\"content\": \"...\"}"`).
- Escape all quotes inside JSON.

### Problems faced:
- Bash did not interpolate variables correctly inside `-d "{\"$VAR\":...}` JSON.
- Result: Discord received the message but all fields like Quality Gate, Coverage, etc were empty.

## Root Cause Discovered

| Root Cause | Impact |
|:---|:---|
| GitHub Actions YAML rules conflict with Bash multi-line `cat <<EOF` syntax | Needed reformatting |
| Bash inside `-d "..."` doesn't expand variables unless managed carefully | Empty metrics |
| We were running on SonarQube CE (Community Edition), not full SonarCloud | Many metrics like coverage, hotspots, duplications were missing |

## Final Working Solution

We decided to simplify the goal:
- After Sonar scan finishes,
- Just send a basic Discord message saying:

```
Analysis Completed. Please check: [project link]
```

### Updated Workflow Behavior:
- Wait for SonarCloud analysis to finalize (poll API 10 times with 5-second gaps).
- If analysis succeeds, send a clean notification to Discord.
- No fetching coverage, critical issues, or other metrics.

## Final GitHub Actions Structure

| Step | Action |
|:---|:---|
| Checkout code | Done |
| Setup Java | Done |
| Run Maven verify + Sonar | Done |
| Wait for SonarCloud Analysis to Complete | Done |
| Send simple Discord alert | Done |

## Lessons Learned

| Lesson | Learning |
|:---|:---|
| GitHub Actions `run: |` block needs strict YAML formatting | Cannot mess up indentation or use raw Bash tricks |
| `cat <<EOF` tricks inside YAML are dangerous | Safer to avoid multi-line Bash unless necessary |
| Bash inside `-d "{\"$VAR\":...}` JSON must manage variable expansions carefully | Or variables will not expand |
| SonarQube CE lacks a lot of analysis data | Cannot rely on coverage, security hotspot info unless configured manually |
| Sometimes simplifying the goal makes systems more reliable | We pivoted from full metrics to simple notification |

## Final Result

- SonarCloud (or SonarQube CE) scan runs successfully.
- Discord notification is sent immediately after analysis finishes.
- No YAML errors, no Bash crashes, no Discord formatting issues.

## End of Devlog

## What We Can Improve in the Future

- If we migrate to SonarCloud (hosted version) or SonarQube Enterprise later, we can re-enable detailed metrics.
- We could send separate green or red Discord embeds depending on Quality Gate status.
- We could include build time, branch name, and commit info inside the Discord message.

## In Short

This journey showed real-world DevOps struggles: Writing correct YAML, handling Bash quirks, understanding Sonar tool limitations, and pivoting design based on reality.


