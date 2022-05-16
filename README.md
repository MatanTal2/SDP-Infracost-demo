# SDP-Infracost-demo

## install infracost

in the [Infracost/docs](https://www.infracost.io/docs/) you can find the instruction for how to install Infracost and how to integrat it to your CI/CD.

### Show case for Infracost command

follow the example for diff and breakdown.

## Integrate to Github Action

### Adding the `INFRACOST_API_KEY` to Github Action secret

- Retrieve your Infracost API key by running `infracost configure get api_key`.
    -- if you didn't register yet please follow the *Get Started* instruction in the [Infracost/docs](https://www.infracost.io/docs/).
- go to *stteinng -> Secrets -> actions*
- Click on *New repository secret*
- Create a repo secret called `INFRACOST_API_KEY` with your API key.
- Create a new file in .github/workflows/infracost.yml in your repo with the following content.

```ymal
# The GitHub Actions docs (https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#on)
# describe other options for 'on', 'pull_request' is a good default.
on: [pull_request]
jobs:
  infracost:
    runs-on: ubuntu-latest # The following are JavaScript actions (not Docker)
    env:
      working-directory: PATH/TO/TERRAFORM/CODE # Update this!

    name: Run Infracost
    steps:
      - name: Check out repository
        uses: actions/checkout@v2

      # Typically the Infracost actions will be used in conjunction with
      # https://github.com/hashicorp/setup-terraform. Subsequent steps
      # can run Terraform commands as they would in the shell.
      - name: Install terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_wrapper: false # This is recommended so the `terraform show` command outputs valid JSON

      # IMPORTANT: add any required steps here to setup cloud credentials so Terraform can run

      - name: Terraform init
        run: terraform init
        working-directory: ${{ env.working-directory }}

      - name: Terraform plan
        run: terraform plan -out tfplan.binary
        working-directory: ${{ env.working-directory }}

      - name: Terraform show
        run: terraform show -json tfplan.binary > plan.json
        working-directory: ${{ env.working-directory }}

      # Install the Infracost CLI, see https://github.com/infracost/actions/tree/master/setup
      # for other inputs such as version, and pricing-api-endpoint (for self-hosted users).
      - name: Setup Infracost
        uses: infracost/actions/setup@v1
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      # Generate Infracost JSON output, the following docs might be useful:
      # Multi-project/workspaces: https://www.infracost.io/docs/features/config_file
      # Combine Infracost JSON files: https://www.infracost.io/docs/features/cli_commands/#combined-output-formats
      - name: Generate Infracost JSON
        run: infracost breakdown --path plan.json --format json --out-file /tmp/infracost.json
        working-directory: ${{ env.working-directory }}
        # Env vars can be set using the usual GitHub Actions syntax
        # See the list of supported Infracost env vars here: https://www.infracost.io/docs/integrations/environment_variables/
        # env:
        #   MY_ENV: ${{ secrets.MY_ENV }}

      # See https://www.infracost.io/docs/features/cli_commands/#comment-on-pull-requests for other options.
      - name: Post Infracost comment
        run: |
          # Posts a comment to the PR using the 'update' behavior.
          # This creates a single comment and updates it. The "quietest" option.
          # The other valid behaviors are:
          #   delete-and-new - Delete previous comments and create a new one.
          #   hide-and-new - Minimize previous comments and create a new one.
          #   new - Create a new cost estimate comment on every push.
          infracost comment github --path /tmp/infracost.json \
                                   --repo $GITHUB_REPOSITORY \
                                   --github-token ${{github.token}} \
                                   --pull-request ${{github.event.pull_request.number}} \
                                   --behavior update
```
