# PSGA Reusable Workflows

PSGA shared and reusable GitHub Action workflows.

---

## EC2 Ephemeral Runners

The following workflows are intended to start an EC2 instance where the main workflow job can run and then terminate the same instance when finished.

- [ec2-runner-start.yaml](.github/workflows/ec2-runner-start.yaml)
- [ec2-runner-stop.yaml](.github/workflows/ec2-runner-stop.yaml)

### Requirements

The following resources are required to be able to run the workflows.

1. OpenID Connect (OIDC) role to authenticate and exchange a short-lived token.
2. GitHub Personal Access Token (PAT) preferably from a machine/system user.
3. VPC with required subnets.
4. Security Group.
5. IAM role name to attach to the created EC2 runner.

All secrets and variables are added to all repositories where these workflows are going to be used. If the repository uses environments, they will need to be populated on each of them. The default branch will read them from the default secrets and variables under security settings.

Refer to the below table for the required inputs and secrets on the caller workflow:

> [!NOTE]
> **Environment Type** = `env_type` (`nonprod` / `prod`).


| Name | Type | Overview |
|------|------|----------|
| `role-to-assume` | secrets | OpenID Connect (OIDC) role (psga-`$env_type`-gh-actions-admin) `arn` created in `psga-terraform` project. |
| `github-token` | secrets |  Personal Access Token (PAT) from GitHub user `devopscongenica` added to the repository access section under settings with `admin` role permission. |
| `aws-region` | vars | The AWS region where the instance is going to be provisioned, currently `eu-west-2`. |
| `ec2-instance-type` | vars | The EC2 instance type to be used, currently `c5.xlarge`. |
| `subnet-id`| vars | The VPC subnet ID where the instance is going to be deployed to, depends on the available availability zones (az) configured on the VPC. |
| `security-group-id`| vars | The security group ID created for the runner commonly named (`$env_type`-gh-runner). |
| `iam-role-name`| vars | The IAM role name created for the runner commonly (`$env_type`-gh-runner). |
| `env_type` | input | Carried over from the caller workflow on PSGA repositories. |
| `label` | output | Default value carried from the start workflow. |
| `ec2-instance-id` | output | Default value carried from the start workflow. |


### Calling Start Runner Workflow

```yaml
start-runner:
    uses: Congenica/psga-reusable-workflows/.github/workflows/ec2-runner-start.yaml@main
    secrets:
      role-to-assume: ${{ secrets.CICD_OIDC_ROLE }}
      github-token: ${{ secrets.GH_PSGA_SYSTEM_PAT }}
    with:
      aws-region: ${{ vars.AWS_REGION }}
      ec2-instance-type: ${{ vars.AWS_INSTANCE_TYPE }}
      subnet-id: ${{ vars.GH_RUNNER_SUBNET_ID }}
      security-group-id: ${{ vars.GH_RUNNER_SECURITY_GROUP_ID }}
      iam-role-name: ${{ vars.GH_RUNNER_IAM_ROLE_NAME }}
      env_type: ${{ inputs.env_type }}
```

### Calling Stop Runner Workflow

```yaml
stop-runner:
    if: ${{ always() }} # required to stop the runner even if previous jobs failed or are cancelled
    needs:
      - start-runner # required to get output from the start-runner job
      - name_of_main_jobs # required to wait when the main job/jobs is/are done.
    uses: Congenica/psga-reusable-workflows/.github/workflows/ec2-runner-stop.yaml@main
    secrets:
      role-to-assume: ${{ secrets.CICD_OIDC_ROLE }}
      github-token: ${{ secrets.GH_PSGA_SYSTEM_PAT }}
    with:
      aws-region: ${{ vars.AWS_REGION }}
      label: ${{ needs.start-runner.outputs.label }}
      ec2-instance-id: ${{ needs.start-runner.outputs.ec2-instance-id }}
```

---

## Reference Data Sync

This workflow syncs reference data which is described using a reference-data-description.yml file with a static data bucket. See [Refence Data Overview](https://congenica.atlassian.net/wiki/spaces/PSG/pages/3055115/Reference+Data+Overview) for an overview of the design. This element handles copying the data which has been labelled with a release branch to the S3 static data bucket

- [reference-data-sync.yaml](.github/workflows/reference-data-sync.yaml) 

### Usage

The Congenica organisation should have all necessary tokens etc necessary to run the workflows. The workflow does use the empheral 
runners system so needs the access that those workflows.

The runners will need access to DVC buckets and the static data bucket. Normally the DVC bucket is the issue as new ones can be added for storing reference data.

### Calling Reference Data Sync Workflow

```yaml
name: Sync to non-prod S3
on:
  push:
    branches:
      - 'prerelease/*'
      - 'release/*'

jobs:
  sync_to_s3_non_prod:
    name: Sync to S3 (non-prod)
    uses: Congenica/psga-reusable-workflows/.github/workflows/reference-data-sync.yaml@main
    secrets:
      CICD_OIDC_ROLE: ${{ secrets.CICD_OIDC_ROLE }}
      GH_PSGA_SYSTEM_PAT: ${{ secrets.GH_PSGA_SYSTEM_PAT }}
      PYPI_USERNAME: ${{ secrets.ARTIFACTORY_DEV_USERNAME }}
      PYPI_PASSWORD: ${{ secrets.ARTIFACTORY_DEV_TOKEN }}
    with:
      destination-bucket: "psga-nonprod-static-data"
      env_type: nonprod
```

---
