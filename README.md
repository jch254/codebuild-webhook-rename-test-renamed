# CodeBuild Webhook Rename Repro

Minimal reproduction of an issue with `aws_codebuild_webhook` where renaming a GitHub repository breaks the webhook without any detectable Terraform drift.

## Summary

Renaming a GitHub repository breaks the webhook created by AWS CodeBuild.

- CodeBuild stops triggering builds
- Terraform reports no drift (`plan -refresh-only`)
- AWS still shows the webhook as configured
- Recreating the webhook fixes the issue

## Steps

1. Create CodeBuild project + webhook via Terraform
2. Confirm builds trigger on push
3. Rename this repository
4. Push again → no builds triggered
5. Run `terraform plan -refresh-only` → no changes detected

## Expected

- Missing or invalid webhook should be detected as drift  
  OR  
- Webhook should be recreated automatically

## Actual

- No drift detected
- CodeBuild no longer triggers builds
- Integration appears healthy from AWS/Terraform

## Notes

- The webhook is created by CodeBuild but not fully reconciled
- Repo rename causes the webhook to be removed or invalidated on GitHub
- AWS / Terraform do not detect this change

## Related issue

<TBD>