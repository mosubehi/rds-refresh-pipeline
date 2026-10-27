
def buildSnapshotId(String prefix, String buildNumber) {
    def safeBuildNumber = buildNumber.replaceAll('[^A-Za-z0-9-]', '-')
    return "${prefix}-${safeBuildNumber}".toLowerCase()
}

node {
    // SAFETY: Keep true for this assessment.
    // Live AWS operations must not run in the lab.
    def DRY_RUN = true

    def config = [
        sourceDbInstance  : 'CHANGE_ME_SOURCE_DB',
        targetDbInstance  : 'CHANGE_ME_LOWER_DB',
        targetAccountId   : 'CHANGE_ME_TARGET_ACCOUNT',
        awsRegion         : 'us-east-1',

        sourceCredentialsId: 'aws-lab-credentials',
        targetCredentialsId: 'aws-lab-credentials',

        snapshotPrefix    : 'rds-refresh',
        targetSnapshotPrefix: 'rds-refresh-copy',

        // Required for a real restore; configure before live use.
        dbInstanceClass   : 'db.t3.micro',
        dbSubnetGroup     : 'CHANGE_ME_SUBNET_GROUP',
        vpcSecurityGroupIds: 'CHANGE_ME_SECURITY_GROUP_ID',

        // Optional target customer-managed KMS key ARN/ID.
        // Required when the copied snapshot needs a target KMS key.
        targetKmsKeyId    : '',

        // Preserve snapshots by default.
        cleanupTemporarySnapshots: false
    ]

    def sourceSnapshotId = ''
    def targetSnapshotId = ''
    def sourceSnapshotArn = ''
    def refreshFailed = false

    try {
        stage('Preflight Checks') {
            echo 'Checking tools and pipeline configuration.'
            sh 'aws --version'

            if (!DRY_RUN) {
                if (config.sourceDbInstance.startsWith('CHANGE_ME_') ||
                    config.targetDbInstance.startsWith('CHANGE_ME_') ||
                    config.targetAccountId.startsWith('CHANGE_ME_') ||
                    config.dbSubnetGroup.startsWith('CHANGE_ME_') ||
                    config.vpcSecurityGroupIds.startsWith('CHANGE_ME_')) {
                    error('Replace all required configuration placeholders before live execution.')
                }

                if (!(config.targetAccountId ==~ /[0-9]{12}/)) {
                    error('Target AWS account ID must contain 12 digits.')
                }

                if (config.sourceDbInstance == config.targetDbInstance) {
                    error('Source and target DB identifiers must be different.')
                }

                if (config.awsRegion !=~ /[a-z]{2}(-gov)?-[a-z]+-[0-9]+/) {
                    error('AWS region format is invalid.')
                }
            }

            sourceSnapshotId = buildSnapshotId(
                config.snapshotPrefix,
                env.BUILD_NUMBER ?: 'local'
            )

            targetSnapshotId = buildSnapshotId(
                config.targetSnapshotPrefix,
                env.BUILD_NUMBER ?: 'local'
            )

            echo "Region: ${config.awsRegion}"
            echo "Source DB: ${config.sourceDbInstance}"
            echo "Target DB: ${config.targetDbInstance}"
            echo "Planned source snapshot: ${sourceSnapshotId}"
            echo "Planned target snapshot: ${targetSnapshotId}"
            echo "Dry run enabled: ${DRY_RUN}"
        }

        stage('Create Source Snapshot') {
            if (DRY_RUN) {
                echo "DRY RUN: Check source DB, create ${sourceSnapshotId} if needed, and wait until available."
            } else {
                withCredentials([
                    usernamePassword(
                        credentialsId: config.sourceCredentialsId,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    withEnv([
                        "AWS_DEFAULT_REGION=${config.awsRegion}",
                        "SOURCE_DB=${config.sourceDbInstance}",
                        "SNAPSHOT_ID=${sourceSnapshotId}"
                    ]) {
                        sh '''
                            set +x
                            set -eu

                            aws rds describe-db-instances \
                                --db-instance-identifier "$SOURCE_DB" \
                                >/dev/null

                            STATUS=$(aws rds describe-db-snapshots \
                                --db-snapshot-identifier "$SNAPSHOT_ID" \
                                --query 'DBSnapshots[0].Status' \
                                --output text 2>/dev/null || true)

                            if [ "$STATUS" = "available" ]; then
                                echo "Existing snapshot is available; reusing it."
                            elif [ "$STATUS" = "None" ] || [ -z "$STATUS" ]; then
                                aws rds create-db-snapshot \
                                    --db-instance-identifier "$SOURCE_DB" \
                                    --db-snapshot-identifier "$SNAPSHOT_ID"
                            else
                                echo "Snapshot exists with status: $STATUS"
                                echo "Refusing to create a duplicate snapshot."
                                exit 1
                            fi

                            timeout 3600 aws rds wait db-snapshot-available \
                                --db-snapshot-identifier "$SNAPSHOT_ID"

                            aws rds describe-db-snapshots \
                                --db-snapshot-identifier "$SNAPSHOT_ID" \
                                --query 'DBSnapshots[0].DBSnapshotArn' \
                                --output text > source-snapshot-arn.txt
                        '''
                    }
                }

                sourceSnapshotArn = readFile('source-snapshot-arn.txt').trim()
                if (!sourceSnapshotArn.startsWith('arn:')) {
                    error('Could not determine the source snapshot ARN.')
                }
            }
        }

        stage('Share Source Snapshot') {
            if (DRY_RUN) {
                echo "DRY RUN: Share snapshot with target account ${config.targetAccountId}."
                echo 'For encrypted snapshots, configure a shareable customer-managed KMS key and its key policy first.'
            } else {
                withCredentials([
                    usernamePassword(
                        credentialsId: config.sourceCredentialsId,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    withEnv([
                        "AWS_DEFAULT_REGION=${config.awsRegion}",
                        "SNAPSHOT_ID=${sourceSnapshotId}",
                        "TARGET_ACCOUNT=${config.targetAccountId}"
                    ]) {
                        sh '''
                            set +x
                            set -eu

                            aws rds modify-db-snapshot-attribute \
                                --db-snapshot-identifier "$SNAPSHOT_ID" \
                                --attribute-name restore \
                                --values-to-add "$TARGET_ACCOUNT"
                        '''
                    }
                }
            }
        }

        stage('Copy Snapshot to Target Account') {
            if (DRY_RUN) {
                echo "DRY RUN: Copy the shared snapshot to ${targetSnapshotId}."
            } else {
                withCredentials([
                    usernamePassword(
                        credentialsId: config.targetCredentialsId,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    withEnv([
                        "AWS_DEFAULT_REGION=${config.awsRegion}",
                        "SOURCE_SNAPSHOT_ARN=${sourceSnapshotArn}",
                        "TARGET_SNAPSHOT_ID=${targetSnapshotId}",
                        "TARGET_KMS_KEY_ID=${config.targetKmsKeyId}"
                    ]) {
                        sh '''
                            set +x
                            set -eu

                            STATUS=$(aws rds describe-db-snapshots \
                                --db-snapshot-identifier "$TARGET_SNAPSHOT_ID" \
                                --query 'DBSnapshots[0].Status' \
                                --output text 2>/dev/null || true)

                            if [ "$STATUS" = "available" ]; then
                                echo "Target snapshot already available; reusing it."
                            elif [ "$STATUS" = "None" ] || [ -z "$STATUS" ]; then
                                if [ -n "$TARGET_KMS_KEY_ID" ]; then
                                    aws rds copy-db-snapshot \
                                        --source-db-snapshot-identifier "$SOURCE_SNAPSHOT_ARN" \
                                        --target-db-snapshot-identifier "$TARGET_SNAPSHOT_ID" \
                                        --kms-key-id "$TARGET_KMS_KEY_ID"
                                else
                                    aws rds copy-db-snapshot \
                                        --source-db-snapshot-identifier "$SOURCE_SNAPSHOT_ARN" \
                                        --target-db-snapshot-identifier "$TARGET_SNAPSHOT_ID"
                                fi
                            else
                                echo "Target snapshot is in unexpected status: $STATUS"
                                exit 1
                            fi

                            timeout 3600 aws rds wait db-snapshot-available \
                                --db-snapshot-identifier "$TARGET_SNAPSHOT_ID"
                        '''
                    }
                }
            }
        }

        stage('Restore Lower-Environment RDS') {
            if (DRY_RUN) {
                echo "DRY RUN: Restore ${config.targetDbInstance} from ${targetSnapshotId}."
            } else {
                withCredentials([
                    usernamePassword(
                        credentialsId: config.targetCredentialsId,
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    withEnv([
                        "AWS_DEFAULT_REGION=${config.awsRegion}",
                        "TARGET_DB=${config.targetDbInstance}",
                        "TARGET_SNAPSHOT_ID=${targetSnapshotId}",
                        "DB_INSTANCE_CLASS=${config.dbInstanceClass}",
                        "DB_SUBNET_GROUP=${config.dbSubnetGroup}",
                        "SECURITY_GROUP_IDS=${config.vpcSecurityGroupIds}"
                    ]) {
                        sh '''
                            set +x
                            set -eu

                            if aws rds describe-db-instances \
                                --db-instance-identifier "$TARGET_DB" \
                                >/dev/null 2>&1; then
                                echo "Target DB already exists. Refusing to overwrite it."
                                exit 1
                            fi

                            aws rds restore-db-instance-from-db-snapshot \
                                --db-instance-identifier "$TARGET_DB" \
                                --db-snapshot-identifier "$TARGET_SNAPSHOT_ID" \
                                --db-instance-class "$DB_INSTANCE_CLASS" \
                                --db-subnet-group-name "$DB_SUBNET_GROUP" \
                                --vpc-security-group-ids $SECURITY_GROUP_IDS \
                                --no-publicly-accessible

                            timeout 7200 aws rds wait db-instance-available \
                                --db-instance-identifier "$TARGET_DB"
                        '''
                    }
                }
            }
        }

        stage('Post-Restore Configuration') {
            if (DRY_RUN) {
                echo 'DRY RUN: Would recreate lower-environment users, reapply permissions, sanitize data, and disable production integrations.'
            } else {
                sh 'bash scripts/post_restore.sh'
            }
        }

        stage('Cleanup') {
            if (config.cleanupTemporarySnapshots) {
                echo 'Cleanup is enabled, but deletion logic must be reviewed and scoped to pipeline-owned snapshots before use.'
            } else {
                echo 'Keeping snapshots for audit and recovery; no snapshots will be deleted.'
            }
        }

        stage('Notify Success') {
            echo 'RDS refresh workflow completed successfully.'
            echo "Build: ${env.BUILD_NUMBER}"
        }

    } catch (err) {
        refreshFailed = true
        currentBuild.result = 'FAILURE'
        echo "RDS refresh failed: ${err.getMessage()}"
        throw err
    } finally {
        echo refreshFailed
            ? 'Pipeline ended with a failure. Review the stage logs.'
            : 'Pipeline execution finished.'
    }
}
