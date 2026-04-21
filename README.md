# CodeBuild Webhook Rename Repro

Minimal reproduction of an issue with `aws_codebuild_webhook` where renaming a GitHub repository breaks the webhook without any detectable Terraform drift, and requires manual intervention to restore.

> **Note:** This repo is already in the post-rename state. It was originally created as `codebuild-webhook-rename-test`, renamed to `codebuild-webhook-rename-test-updated`, and renamed again to `codebuild-webhook-rename-test-renamed-again`. The steps below describe the full sequence from the beginning.

---

## Summary

Renaming a GitHub repository breaks the webhook created by AWS CodeBuild.

- CodeBuild stops triggering builds
- Terraform reports no drift (`plan -refresh-only`)
- Updating the repo URL alone does not fix the issue
- Webhook must be explicitly recreated

---

## Lifecycle of the Issue

### 1. Initial state (working)

- CodeBuild project + webhook created via Terraform
- GitHub webhook exists
- Push events trigger builds ✅

---

### 2. Repository rename

- `codebuild-webhook-rename-test` renamed to `codebuild-webhook-rename-test-updated`, then renamed again to `codebuild-webhook-rename-test-renamed-again`
- GitHub removes or invalidates the webhook
- CodeBuild no longer receives events ❌

Observations:

- No errors in AWS
- No errors in Terraform
- Git operations continue working (redirects)
- Integration appears healthy

---

### 3. Terraform refresh

terraform plan -refresh-only

Result:

- ❌ No changes detected
- Terraform still believes webhook exists

---

### 4. Update repository source (partial fix)

source {
  type     = "GITHUB"
  location = var.repo_url
}

terraform apply

Result:

- CodeBuild project updates to new repo
- ❌ Webhook is NOT recreated
- ❌ Builds still do not trigger

---

## The Fix (Required)

Updating the source location alone is NOT sufficient.

You must force recreation of the webhook:

terraform destroy -target=aws_codebuild_webhook.example
terraform apply   -target=aws_codebuild_webhook.example

Result:

- Webhook recreated in GitHub
- Integration restored
- Builds trigger again ✅

---

## Steps to Reproduce

1. Create a GitHub repo (e.g. `codebuild-webhook-rename-test`)
2. Set `repo_url` in `terraform.tfvars` to the original repo URL
3. Apply Terraform to create:
   - CodeBuild project
   - `aws_codebuild_webhook`
4. Push commit → verify build triggers
5. Rename the GitHub repository (e.g. to `codebuild-webhook-rename-test-updated`)
6. Push commit → no builds triggered
7. Run `terraform plan -refresh-only` → no drift detected
8. Update `repo_url` in `terraform.tfvars` to the new URL → `terraform apply`
9. Push commit → still no builds triggered
10. Recreate webhook: `terraform destroy -target=aws_codebuild_webhook.example && terraform apply -target=aws_codebuild_webhook.example` → builds resume

---

## Expected

- Missing or invalid webhook should be detected as drift  
OR  
- Webhook should be recreated when repository source changes  

---

## Actual

- No drift detected after webhook is removed/invalidated
- Updating repository source does not recreate webhook
- CodeBuild stops triggering builds silently
- Integration appears healthy from AWS/Terraform

---

## Notes

- GitHub repo rename removes or breaks the webhook
- GitHub redirects do not propagate to integrations
- Terraform state still believes webhook exists
- Provider does not detect missing webhook
- Webhook lifecycle is effectively not reconciled after creation

---

## Public Build History

[https://ap-southeast-2.codebuild.aws.amazon.com/project/eyJlbmNyeXB0ZWREYXRhIjoidmZkREJWTCtKZFJzK3B6RUxtTWlYUDFrQjg3RGl5ajJGNXVMU3M5ZzJwaUptTUEwcUM0THovZk9TeDhpV3NyaUtiQk5WYm9tNVhrZmxJZCtIeXFvUGQxRUNiV0VHS2c4T200Z0VKTWJ4RWRsMi9LQTQ1YUNLR2d1ZFgveEpHYXNCVnpoM1E9PSIsIml2UGFyYW1ldGVyU3BlYyI6ImFwckVBQTBMdFhFZ2t6RFAiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D](https://ap-southeast-2.codebuild.aws.amazon.com/project/eyJlbmNyeXB0ZWREYXRhIjoidmZkREJWTCtKZFJzK3B6RUxtTWlYUDFrQjg3RGl5ajJGNXVMU3M5ZzJwaUptTUEwcUM0THovZk9TeDhpV3NyaUtiQk5WYm9tNVhrZmxJZCtIeXFvUGQxRUNiV0VHS2c4T200Z0VKTWJ4RWRsMi9LQTQ1YUNLR2d1ZFgveEpHYXNCVnpoM1E9PSIsIml2UGFyYW1ldGVyU3BlYyI6ImFwckVBQTBMdFhFZ2t6RFAiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D)

---

## Triggering Builds (for testing)

git commit --allow-empty -m "trigger build"
git push

Expected behavior:

| State                     | Build triggers |
|--------------------------|---------------|
| Before rename            | ✅            |
| After rename             | ❌            |
| After repo URL update    | ❌            |
| After webhook recreate   | ✅            |

---

## Related issue

TBC