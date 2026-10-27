
# RDS Database Refresh Pipeline

## 1. Overview

This project describes a Jenkins Scripted Pipeline for refreshing a lower-environment Amazon RDS database from a source database snapshot.

The workflow includes snapshot creation, cross-account sharing, snapshot copying, target database restoration, post-restore configuration, cleanup, and notification.

The assessment is designed for dry-run validation. No AWS resources or database changes are required to validate the workflow.

## 2. Project Structure

```text
rds-refresh-pipeline/
├── Jenkinsfile
├── README.md
├── architecture.mmd
└── scripts/
    └── post_restore.sh
```

## 3. Workflow

1. Validate configuration and required tools.
2. Create a snapshot of the source RDS instance.
3. Share the snapshot with the target AWS account.
4. Copy the shared snapshot into the target account.
5. Restore the lower-environment RDS instance.
6. Apply lower-environment configuration.
7. Clean up temporary resources where explicitly permitted.
8. Report success or failure.

## 4. Prerequisites

- Jenkins with Scripted Pipeline support.
- AWS CLI installed in the Jenkins execution environment.
- AWS credentials stored in Jenkins Credentials.
- Appropriate IAM permissions for the intended AWS operations.
- Source and target AWS account IDs and an AWS Region.
- Target VPC, subnet group, security group, and database configuration for a real deployment.

No live AWS resources are required for dry-run validation.

## 5. Configuration and Credentials

The Jenkinsfile contains non-secret configuration values, including database identifiers, AWS Region, snapshot prefixes, and the target account ID.

AWS credentials must be stored in Jenkins Credentials and referenced by credential ID. Access keys and passwords must never be hardcoded in the repository or printed in build logs.

For production deployments, prefer IAM roles and temporary credentials where supported.

## 6. Cross-Account Sharing and Encryption

For cross-account snapshot sharing, the source account must grant the target account permission to restore the snapshot.

If the snapshot is encrypted, the workflow must also use an appropriate customer-managed AWS KMS key and configure the required key policy and IAM permissions.

The AWS-managed RDS KMS key cannot be shared for cross-account snapshot restoration. The target account may need to copy the shared snapshot using a target-account KMS key.

## 7. Post-Restore Configuration

The `scripts/post_restore.sh` script currently operates in dry-run mode. It lists the intended tasks without connecting to a database.

The intended post-restore activities are:

- Recreate lower-environment database users.
- Reapply lower-environment permissions.
- Apply environment-specific configuration.
- Sanitize sensitive production data according to policy.
- Disable production integrations and scheduled jobs.
- Verify the resulting lower-environment configuration.

Database-specific implementation requires confirmation of the database engine and environment requirements.

## 8. Safety, Error Handling, and Idempotency

The assessment must be validated without creating or deleting AWS resources.

A production-ready implementation should:

- Validate inputs before performing operations.
- Use deterministic, collision-resistant snapshot identifiers.
- Check resource state before creating or reusing resources.
- Wait for snapshots and database instances to reach the required states, with timeouts.
- Stop on failures and preserve the original error.
- Avoid overwriting an existing target database.
- Delete only explicitly identified temporary resources.
- Retain snapshots by default unless cleanup is explicitly approved.

## 9. Validation

Validate the post-restore shell script:

```bash
bash -n scripts/post_restore.sh
./scripts/post_restore.sh
```

Review the Jenkinsfile for Scripted Pipeline syntax and confirm that dry-run mode is enabled before running any validation build.

Dry-run output confirms the planned workflow only. It does not prove that AWS snapshot, sharing, copying, or restoration operations have executed successfully.

## 10. Assumptions and Limitations

- The database engine has not been specified by the assessment.
- No source RDS instance is available for live testing.
- The pipeline is intended for dry-run/static validation.
- Cross-account permissions and KMS configuration depend on the actual AWS environment.
- Live execution requires environment-specific review and testing before use.
