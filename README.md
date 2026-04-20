# CodeBuild Webhook Rename Repro

Minimal reproduction of an issue with `aws_codebuild_webhook` where renaming a GitHub repository breaks the webhook without any detectable Terraform drift.

## Summary

Renaming a GitHub repository breaks the webhook created by AWS CodeBuild.

- CodeBuild stops triggering builds
- Terraform reports no drift (`plan -refresh-only`)
- AWS still shows the webhook as configured
- Updating the repo URL in Terraform does not recreate the webhook
- Recreating the webhook manually fixes the issue

---

## Steps

1. Create CodeBuild project + webhook via Terraform
2. Confirm builds trigger on push
3. Rename this repository
4. Push again → no builds triggered
5. Run `terraform plan -refresh-only` → no changes detected
6. Update `repo_url` in Terraform to match renamed repo → apply
7. Observe webhook is still missing and not recreated

---

## Expected

- Missing or invalid webhook should be detected as drift  
  OR  
- Webhook should be recreated automatically when source changes  

---

## Actual

- No drift detected after webhook becomes invalid
- Updating the repository URL does not recreate the webhook
- CodeBuild no longer triggers builds
- Integration appears healthy from AWS/Terraform

---

## Notes

- The webhook is created by CodeBuild but not fully reconciled
- Repo rename causes the webhook to become invalid or disappear on GitHub
- Updating CodeBuild source does not restore the webhook
- AWS / Terraform do not detect this change

---

## Triggering Builds (for testing)

To verify behavior, push commits to this repo:

```bash
git commit --allow-empty -m "trigger build"
git push