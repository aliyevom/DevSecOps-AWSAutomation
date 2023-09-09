# Implementing DevSecOps on AWS with CodeCommit, CodeBuild, CodePipeline, and CloudFormation

This comprehensive guide demonstrates how to establish a multi-account DevSecOps CodePipeline on AWS.

To safeguard your development operations (DevOps) supply chain, it is crucial to minimize your potential risks and enforce strict separation of duties. Therefore, it is highly advisable to employ multiple AWS accounts for your pipelines. Typically, this involves setting up one development account for your developers, which houses your CodeCommit repositories. Then, you should have a separate account for your DevOps tools, such as your build server, pipeline, artifact bucket, and so forth. Lastly, the remaining accounts are designated for various environments, such as testing, production, and others.

For enhanced security, we employ manual approvals in both the DevOps account and the target accounts. However, if your specific use case permits, you can opt to automate this process entirely.

![Manual Approval](https://securitydude-article-images.s3.amazonaws.com/manual-approval.png)

Here is an overview of our architecture:
![Architecture](https://securitydude-article-images.s3.amazonaws.com/devops-arch1.png)

Let's get started:

1. Create a reference table like the one below:

| Field             | Value                 |
| ----------------- | --------------------- |
| Project Name      | test_pipeline         |
| Dev Acct #        | 111111111111          |
| Tools (DevOps) Acct # | 222222222222      |
| Target Acct #(s)  | 333333333333,444444444444 |

I recommend using AWS Single Sign-On (SSO) to simplify switching between accounts. Optionally, you can use 1 - 3+ Organizational Units (OUs) for this purpose.

2. Create the three stacks in the following order (use us-east-1 region). Wait for each stack to complete fully before proceeding to the next. If you have multiple target accounts, run the target stack in each of them. (Note that currently only US-East-1 is supported.) You don't need to clone this repository to launch the stacks; simply click the "Launch Stack" buttons.

| Order | Stack Name                  | Launch                                                                                                                            |
| ----- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 1     | Dev Account Stack           | [![Dev Account Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=dev-stack&templateURL=https://securitydude-pipeline-demo.s3.amazonaws.com/dev-templates/pipeline-MASTER.yaml) |
| 2     | DevOps (Tools) Account Stack | [![DevOps (Tools) Account Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=devops-stack&templateURL=https://securitydude-pipeline-demo.s3.amazonaws.com/devops-templates/pipeline-MASTER.yaml) |
| 3     | Target Account Stack(s)     | [![Target Account Stack(s)](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=target-stack&templateURL=https://securitydude-pipeline-demo.s3.amazonaws.com/target-templates/pipeline-MASTER.yaml) |

3. Log into the Dev account to create a CodeCommit user:

   - Note the CodeCommit repository name.
   - Go to [IAM Users](https://console.aws.amazon.com/iam/home?region=us-east-1#/users) in the AWS Management Console.
   - Add a user with the appropriate permissions.
   - On the user summary, navigate to the "Security credentials" tab.
   - Click the "Upload SSH public key" button and add your public key (e.g., id_rsa.pub). Make a note of the SSH key ID.

   Update your .ssh/config file as shown below (replace `<SSH Key ID>` with the actual key ID):

   ```shell
   Host git-codecommit.*.amazonaws.com
   User ...
   IdentityFile ~/.ssh/id_rsa
   ```

   - Clone the CodeCommit repository and create a "testing" branch:

   ```shell
   git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/<repo name>
   cd <repo name>
   wget https://securitydude-pipeline-demo.s3.amazonaws.com/stack-for-testing/pipeline-testing.zip
   unzip pipeline-testing.zip
   git checkout -b testing
   git add *.yaml
   git commit -m "Initial Commit"
   git push origin testing
   ```

4. Go to the DevOps (Tools) Account and check CodePipeline. The pipeline may take up to 2-3 minutes to start.

   ![CodeCommit](https://securitydude-article-images.s3.amazonaws.com/codecommit.png)

   - The Source and Build stages should succeed (note that your commit message will appear in the "SourceAction"). Afterward, manually approve the deployment.

   ![CodeBuild](https://securitydude-article-images.s3.amazonaws.com/codebuild.png)

   - The next stage is the "Release Lambda," which pushes the repository files to the S3 bucket in each of the target accounts. The Lambda execution should succeed.

5. Now, go to the target accounts and view CodePipeline.

   - Again, manually approve the deployment.
   - Now, CloudFormation will run.
   - Go to CloudFormation and view the resources it has created.
   - Repeat this process for any additional target accounts.

   ![Target Account Pipeline](https://securitydude-article-images.s3.amazonaws.com/target-account-pipeline.png)

6. For testing purposes:

   Remove one of the cfn_nag suppression lines from `sg1.yaml`.
   Commit the change to the repository.
   A new pipeline run will be triggered.
   It will fail at the CodeBuild step.
   Go to Build --> Report history and click on the latest report to view the results.

   ![CodeBuild Report](https://securitydude-article-images.s3.amazonaws.com/codebuild-report.png)

7. CodeBuild also checks for valid template syntax. For example, if you remove a colon (`:`) anywhere in the file and commit the change, CodeBuild will fail, and the flawed template will not be released to the test or production environments.

   ![CodeBuild Validation](https://securitydude-article-images.s3.amazonaws.com/codebuild-validate.png)

Security Tip: Remove IAM User (root) permissions in the KMS stacks and replace them with your key administrator principal. There are two KMS stacks, one in the tools account and one in the target account(s).

(c) Copyright 2023 - Aliyev Omar

**Disclaimer:**
For informational and educational purposes only. Bugs may exist and can be reported on GitHub. Note that using this guide will incur AWS charges.