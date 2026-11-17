# Security Policy

## Disclaimer

This repository contains sample code for demonstration and learning. It is **not production-ready**.
Review and test it against your own security, compliance and operational requirements before any
production use. The authors and AWS make no warranty regarding its fitness for production workloads.

## Reporting Vulnerabilities

If you discover a potential security issue in this project we ask that you notify AWS/Amazon Security
via our [vulnerability reporting page](https://aws.amazon.com/security/vulnerability-reporting/).
Please do **not** create a public GitHub issue.

## AWS Services Used

AWS Step Functions, AWS Lambda (Python 3.13), Amazon DynamoDB, AWS KMS, Amazon S3 (template bucket, deployment
artifact bucket and its access log bucket), AWS CloudFormation, AWS Systems Manager Parameter Store, AWS IAM,
Amazon CloudWatch Logs and AWS X-Ray. The pipelines that the solution creates use AWS CodeCommit, AWS CodeBuild and
AWS CodePipeline.

## Prerequisites and Permissions

Deploying `template.yml` requires permissions to create IAM roles and managed policies, a KMS key, a DynamoDB table,
a Lambda function, a Step Functions state machine, a log group and SSM parameters.

The **Lambda function role** can list and read `samplecode/*` and read `input-references/*` in the template bucket, and
write `generated-artifacts/*`. The function has no network or credentials of its own beyond that, and installs no
packages at run time.

The **state machine role** can write to the DynamoDB table, invoke the function, describe, create and update only
the two kinds of stacks of this solution, and only with the two template URLs of the template bucket
(`cloudformation:TemplateUrl` condition). CloudFormation also uses this role when it creates the stacks, so the
role has the permissions to create exactly the resources in those stacks:

- IAM: create and manage only `Codebuild-Role-<Region>`, `Codepipeline-Role-<Region>` and the two managed policies
  `codebuild-policy-<Region>` and `codepipeline-policy-<Region>`. It can attach only those two policies to those two
  roles, and pass only those roles to CodeBuild and CodePipeline. It has no user-policy or inline-policy actions.
- CodeCommit, CodeBuild and CodePipeline: only resources named `*-Repo`, `*-Build` and `*-Pipeline`.
- S3: only buckets named `deployment-artifact-bucket*`. SSM: only the named parameters.

The **CodeBuild role** can read and write the deployment artifact bucket, write its own build logs and upload
test reports. It has no IAM, SSM or CodeCommit permissions. The **CodePipeline role** can read the repositories,
start the build projects and use the artifact bucket.

## Known Security Considerations

The following items were reviewed and accepted for this sample. Revisit them before any production use.

| Item | Why it is acceptable here | Production recommendation |
|------|---------------------------|---------------------------|
| The state machine role can create IAM roles and policies (limited to four fixed names, with conditions) | CloudFormation needs this to create the dependencies stack | Create the dependencies stack once with an administrator, and remove the IAM actions from the state machine role. Or add a permissions boundary and require it on the roles |
| The log delivery actions on the state machine role and `codecommit:ListRepositories` use `Resource: "*"` | These actions do not support resource-level permissions | Keep, and review them regularly |
| Anyone who can start the state machine can create pipelines and CodeCommit repositories in the account | The state machine validates its input, but does not check who calls it | Allow `states:StartExecution` only for a small group, and consider an approval step |
| Fixed IAM role and policy names, so only one deployment per account and Region | The state machine role is limited to these names | Add a naming parameter if you need more |
| Lambda is not in a VPC, has no dead-letter queue and no reserved concurrency | The function is invoked synchronously by Step Functions, calls only Amazon S3 and runs for a few seconds | Add a VPC configuration with a gateway endpoint for Amazon S3 if your policy needs it |
| S3 Object Lock is not turned on for the artifact bucket | Object Lock would block the clean-up steps for this sample | Turn it on if you must keep artifacts immutable |
| The artifact bucket uses SSE-S3, not a customer managed key | CodePipeline artifacts and build outputs are sample data. S3 access logs can't be delivered to an SSE-KMS bucket | Use a customer managed key for the artifact bucket and send logs to a separate bucket that uses SSE-S3 |
| `PollForSourceChanges` is `true` in the generated pipelines | Simple, and works without extra resources | Use event-based change detection |
| The template bucket is created and secured by you, not by this solution | The solution only reads and writes the prefixes listed above | Block public access, turn on versioning, encryption and TLS-only access for that bucket too |

## Operational Notes

- The template bucket contains the buildspec fragments and the sample code. Anyone who can write to it can change what
  the generated pipelines run. Restrict write access to the bucket.
- The generated `buildspec.yml` runs with the CodeBuild role permissions. Don't add secrets to the fragments.
- DynamoDB items and state machine logs contain the project name and tech stack only. Execution data is not logged.
- Delete the pipeline stacks you no longer need. CodeBuild report groups and log groups stay after a stack is deleted.

## Resource Cleanup

See the "Clean up" section of the README.

## Dependencies

The Lambda function uses only the Python standard library and the AWS SDK for Python (Boto3) that is provided by the
Lambda runtime. The Python sample installs `pytest` at a pinned version. The Java sample uses Maven dependencies with
pinned versions. Update them regularly.
