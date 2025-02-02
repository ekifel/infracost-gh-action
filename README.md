# Infracost GitHub Action

This GitHub Action runs [Infracost](https://infracost.io) against pull requests whenever Terraform files change. It automatically adds a pull request comment showing the cost estimate difference for the planned state if a configurable percentage threshold is crossed. See [this repo for a demo](https://github.com/infracost/gh-actions-demo).

The Action uses the latest version of Infracost by default as we regularly add support for more cloud resources. If you run into any issues, please join our [community Slack channel](https://www.infracost.io/community-chat); we'd be happy to guide you through it.

As mentioned in the [FAQ](https://www.infracost.io/docs/faq), **no** cloud credentials, secrets, tags or resource identifiers are sent to the Cloud Pricing API. That API does not become aware of your cloud spend; it simply returns cloud prices to the CLI so calculations can be done on your machine. Infracost does not make any changes to your Terraform state or cloud resources.

<img src="screenshot.png" width=557 alt="Example screenshot" />

## Inputs

### `path`

**Optional** Path to the Terraform directory or JSON/plan file. Either `path` or `config_file` is required.

### `terraform_plan_flags`

**Optional** Flags to pass to the 'terraform plan' command, e.g. `"-var-file=my.tfvars -var-file=other.tfvars"`. Applicable when path is a Terraform directory.

### `terraform_workspace`

**Optional** The Terraform workspace to use. Applicable when path is a Terraform directory. Only set this for multi-workspace deployments, otherwise it might result in the Terraform error "workspaces not supported".

### `usage_file`

**Optional** Path to Infracost [usage file](https://www.infracost.io/docs/usage_based_resources#infracost-usage-file) that specifies values for usage-based resources, see [this example file](https://github.com/infracost/infracost/blob/master/infracost-usage-example.yml) for the available options.

### `config_file`

**Optional** If your repo has **multiple Terraform projects or workspaces**, define them in a [config file](https://www.infracost.io/docs/config_file/) and set this input to its path. Their results will be combined into the same diff output. Cannot be used with path, terraform_plan_flags or usage_file inputs. 

### `percentage_threshold`

**Optional** The absolute percentage threshold that triggers a pull request comment with the diff. Defaults to 0, meaning that a comment is posted if the cost estimate changes. For example, set to 5 to post a comment if the cost estimate changes by more than plus or minus 5%.

## Environment variables

This section describes the required environment variables. Other supported environment variables are described in the [this page](https://www.infracost.io/docs/integrations/environment_variables). [Repo secrets](https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository) can be used for sensitive environment values.

Terragrunt users should also read [this page](https://www.infracost.io/docs/iac_tools/terragrunt). Terraform Cloud/Enterprise users should also read [this page](https://www.infracost.io/docs/iac_tools/terraform_cloud_enterprise).

### `INFRACOST_API_KEY`

**Required** To get an API key [download Infracost](https://www.infracost.io/docs/#installation) and run `infracost register`.

### `GITHUB_TOKEN`

**Required** GitHub token used to post comments, should be set to `${{ secrets.GITHUB_TOKEN }}` to use the default GitHub token available to actions (see example in the [Usage section](#usage)).

### Cloud credentials

**Required** You do not need to set cloud credentials if you use Terraform Cloud/Enterprise's remote execution mode, instead you should follow [this page](https://www.infracost.io/docs/iac_tools/terraform_cloud_enterprise).

For all other users, the following is needed so Terraform can run `init`:
- AWS users should set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
- GCP users should set `GOOGLE_CREDENTIALS` or read [this section](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference#full-reference) of the Terraform docs for other options.

### `INFRACOST_TERRAFORM_BINARY`

**Optional** Used to change the path to the `terraform` binary or version, see [this page](https://www.infracost.io/docs/integrations/environment_variables/#cicd-integrations) for the available options.

### `GIT_SSH_KEY`

**Optional** If you're using Terraform modules from private Git repositories you can set this environment variable to your private Git SSH key so Terraform can access your module.

## Outputs

### `total_monthly_cost`

The new total monthly cost estimate.

### `past_total_monthly_cost`

The past total monthly cost estimate.

## Usage

1. [Add repo secrets](https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository) for `INFRACOST_API_KEY` and any other required credentials to your GitHub repo (e.g. `AWS_ACCESS_KEY_ID`).

2. Create a new file in `.github/workflows/infracost.yml` in your repo with the following content. Use the Inputs and Environment Variables section above to decide which `env` and `with` options work for your Terraform setup. The following example uses `path` to specify the location of the Terraform directory and `terraform_plan_flags` to specify the variables file to use when running `terraform plan`. The GitHub Actions [docs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#on) describe other options for `on`, though `pull_request` is probably what you want.

  ```
  on:
    pull_request:
      paths:
      - '**.tf'
      - '**.tfvars'
      - '**.tfvars.json'
  jobs:
    infracost:
      runs-on: ubuntu-latest
      name: Show infracost diff
      steps:
      - name: Check out repository
        uses: actions/checkout@v2
      - name: Run infracost diff
        uses: infracost/infracost-gh-action@master # Use a specific version instead of master if locking is preferred
        env:
          INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Do not change
          # See the cloud credentials section for the options
        with:
          entrypoint: /scripts/ci/diff.sh # Do not change
          path: path/to/code
          terraform_plan_flags: -var-file=my.tfvars
  ```

3. Send a new pull request to change something in Terraform that costs money; a comment should be posted on the pull request. Check the GitHub Actions logs and [this page](https://www.infracost.io/docs/integrations/cicd#cicd-troubleshooting) if there are issues.

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
