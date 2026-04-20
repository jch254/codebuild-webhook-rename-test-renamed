# CodeBuild Webhook Rename Repro

Minimal reproduction of an issue with `aws_codebuild_webhook` where renaming a GitHub repository breaks the webhook without any detectable Terraform drift, and requires manual intervention to restore.

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

- GitHub repository is renamed
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

terraform destroy -target=aws_codebuild_webhook.this
terraform apply   -target=aws_codebuild_webhook.this

Result:

- Webhook recreated in GitHub
- Integration restored
- Builds trigger again ✅

---

## Steps to Reproduce

1. Apply Terraform to create:
   - CodeBuild project
   - aws_codebuild_webhook
2. Push commit → verify build triggers
3. Rename the GitHub repository
4. Push commit → no builds triggered
5. Run terraform plan -refresh-only → no drift detected
6. Update repo URL in Terraform → apply
7. Push commit → still no builds triggered
8. Recreate webhook via targeted apply → builds resume

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

[https://ap-southeast-4.codebuild.aws/project/codebuild-webhook-rename-repro](https://ap-southeast-4.codebuild.aws/project/codebuild-webhook-rename-repro)

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

<TBD>