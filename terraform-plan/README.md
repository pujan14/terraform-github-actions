# terraform-plan action

This is one of a suite of terraform related actions - find them at [dflook/terraform-github-actions](https://github.com/dflook/terraform-github-actions).

This actions generates a terraform plan.
If the triggering event relates to a PR it will add a comment on the PR containing the generated plan.

<p align="center">
    <img src="plan.png" width="600">
</p>

The `GITHUB_TOKEN` environment variable is must be set for the PR comment to be added.
The action can be run on other events, which prints the plan to the workflow log.

The [dflook/terraform-apply](https://github.com/dflook/terraform-github-actions/tree/master/terraform-apply) action can be used to apply the generated plan.

## Inputs

* `path`

  Path to the terraform configuration

  - Type: string
  - Required

* `workspace`

  Terraform workspace to run the plan for

  - Type: string
  - Optional
  - Default: `default`

* `label`

  An friendly name for the environment the terraform configuration is for.
  This will be used in the PR comment for easy identification.

  It must be the same as the `label` used in the corresponding `terraform-apply` command.

  - Type: string
  - Optional

* `variables`

  Variables to set for the terraform plan. This should be valid terraform syntax - like a [variable definition file](https://www.terraform.io/docs/language/values/variables.html#variable-definitions-tfvars-files).

  ```yaml
  with:
    variables: |
      image_id = "${{ secrets.AMI_ID }}"
      availability_zone_names = [
        "us-east-1a",
        "us-west-1c",
      ]
  ```

  Variables set here override any given in `var_file`s.

  > :warning: Secret values are not masked in the PR comment. Set a `label` to avoid revealing the variables in the PR.

  - Type: string
  - Optional

* ~~`var`~~

  > :warning: **Deprecated**: Use the `variables` input instead.

  Comma separated list of terraform vars to set.

  This is deprecated due to the following limitations:
  - Only primitive types can be set with `var` - number, bool and string.
  - String values may not contain a comma.
  - Values set with `var` will be overridden by values contained in `var_file`s

  You can change from `var` to `variables` by putting each variable on a separate line and ensuring each string value is quoted.

  For example:
  ```yaml
  with:
    var: instance_type=m5.xlarge,nat_type=instance
  ```
  Becomes:
  ```yaml
  with:
    variables: |
      instance_type="m5.xlarge"
      nat_type="instance"
  ```

  - Type: string
  - Optional

* `var_file`

  Comma separated list of tfvars files to use.
  Paths should be relative to the GitHub Actions workspace

  - Type: string
  - Optional

* `backend_config`

  Comma separated list of terraform backend config values.

  - Type: string
  - Optional

* `backend_config_file`

  Comma separated list of terraform backend config files to use.
  Paths should be relative to the GitHub Actions workspace

  - Type: string
  - Optional

* `parallelism`

  Limit the number of concurrent operations

  - Type: number
  - Optional
  - Default: 10

* `add_github_comment`

  The default is `true`, which adds a comment to the PR with the generated plan.
  Set to `false` to disable the comment - the plan will still appear in the workflow log.

  - Type: bool
  - Optional
  - Default: true

## Environment Variables

* `GITHUB_TOKEN`

  The GitHub authorization token to use to create comments on a PR.
  The token provided by GitHub Actions can be used - it can be passed by
  using the `${{ secrets.GITHUB_TOKEN }}` expression, e.g.

  ```yaml
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ```

  - Type: string
  - Optional

* `TERRAFORM_CLOUD_TOKENS`

  API tokens for terraform cloud hosts, of the form `<host>=<token>`. Multiple tokens may be specified, one per line.
  These tokens may be used with the `remote` backend and for fetching required modules from the registry.

  e.g for terraform cloud:
  ```yaml
  env:
    TERRAFORM_CLOUD_TOKENS: app.terraform.io=${{ secrets.TF_CLOUD_TOKEN }}
  ```

  With Terraform Enterprise or other registries:
  ```yaml
  env:
    TERRAFORM_CLOUD_TOKENS: |
      app.terraform.io=${{ secrets.TF_CLOUD_TOKEN }}
      terraform.example.com=${{ secrets.TF_REGISTRY_TOKEN }}
  ```

  - Type: string
  - Optional

* `TERRAFORM_SSH_KEY`

  A SSH private key that terraform will use to fetch git module sources.

  This should be in PEM format.

  For example:
  ```yaml
  env:
    TERRAFORM_SSH_KEY: ${{ secrets.TERRAFORM_SSH_KEY }}
  ```

  - Type: string
  - Optional

* `TF_PLAN_COLLAPSE_LENGTH`

  When PR comments are enabled, the terraform output is included in a collapsable pane.
  
  If a terraform plan has fewer lines than this value, the pane is expanded
  by default when the comment is displayed.

  ```yaml
  env:
    TF_PLAN_COLLAPSE_LENGTH: 30
  ```

  - Type: integer
  - Optional
  - Default: 10

## Outputs

* `changes`

  Set to 'true' if the plan would apply any changes, 'false' if it wouldn't.

  Note that with terraform <0.13 an apply may still be needed to update any outputs, even if no
  resources would change. With terraform >=0.13 this is correctly set to 'true' whenever an apply
  needs to be run.

## Example usage

### Automatically generating a plan

This example workflow runs on every push to an open pull request,
and create or updates a comment with the terraform plan

```yaml
name: PR Plan

on: [pull_request]

jobs:
  plan:
    runs-on: ubuntu-latest
    name: Create terraform plan
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}            
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform plan
        uses: dflook/terraform-plan@v1
        with:
          path: my-terraform-config
```

### A full example of inputs

This example workflow demonstrates most of the available inputs:
- The environment variables are set at the workflow level.
- The PR comment will be labelled `production`, and the plan will use the `prod` workspace.
- Variables are read from `env/prod.tfvars`, with `turbo_mode` overridden to `true`.
- The backend config is taken from `env/prod.backend`, and the token is set from a secret.

```yaml
name: PR Plan

on: [pull_request]

env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  TERRAFORM_CLOUD_TOKENS: terraform.example.com=${{ secrets.TF_REGISTRY_TOKEN }}
  TERRAFORM_SSH_KEY: ${{ secrets.TERRAFORM_SSH_KEY }}

jobs:
  plan:
    runs-on: ubuntu-latest
    name: Create terraform plan
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform plan
        uses: dflook/terraform-plan@v1
        with:
          path: my-terraform-config
          label: production
          workspace: prod
          var_file: env/prod.tfvars
          variables:
            turbo_mode=true
          backend_config_file: env/prod.backend
          backend_config: token=${{ secrets.BACKEND_TOKEN }}
```

### Generating a plan using a comment

This workflow generates a plan on demand, triggered by someone
commenting `terraform plan` on the PR. The action will create or update
a comment on the PR with the generated plan.

```yaml
name: Terraform Plan

on: [issue_comment]

jobs:
  plan:
    if: github.event.issue.pull_request && contains(github.event.comment.body, 'terraform plan')
    runs-on: ubuntu-latest
    name: Create terraform plan
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: refs/pull/${{ github.event.issue.number }}/merge

      - name: terraform plan
        uses: dflook/terraform-plan@v1
        with:
          path: my-terraform-config
```
