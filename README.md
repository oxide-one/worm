# DigitalOcean based ephemeral VMs

I made this because I wanted to utilize the capabilities of [DigitalOcean](https://digitalocean.com) Droplets to run self hosted runners, without an extraordinary bill at the end of the month.

Use at your own peril. I don't claim to provide any warranty of code quality, but it's working for me.

## Flow

This takes a slightly more manual approach to things, requiring manual steps to start and stop the workflows. 

This is different to the AWS Based one written by [Philips Labs](https://github.com/philips-labs/terraform-aws-github-runner), as you are required to explicitly setup the VM and tear it down for each run. Thanfully it's fairly simple to actually do. Using this approach does allow for a lot more flexibility in how you do things, as you can run multiple workflows on one machine, and then tear it down.

## Usage

First, you're going to need to get a Github PAT with the following permissions:
```
repo
workflow
```

And you will also need to create a [DigitalOcean API key](https://cloud.digitalocean.com/account/api/tokens). These aren't scoped, but if they end up becoming scoped in the future, I'll update the repo.

From there, you'll need to reference this repo in your Workflows.

```yaml
# .github/workflows/your-workflow-name.yaml
jobs:
  spin-up-droplet:
    name: Spin up Droplet
    uses: oxide-one/workflow-runner-maker/.github/workflows/spinup.yaml@main
    with:
      name: gha-${{ github.run_id }}-${{ github.run_number }}
    secrets:
      access-token: ${{ secrets.YOUR_GITHUB_PERSONAL_ACCESS_TOKEN }}
      do-access-token: ${{ secrets.YOUR_DIGITALOCEAN_ACCESS_TOKEN }}
```

This will create a Droplet for you that will register to the repo you triggered it from.

Then, run the rest of your workflow, referencing the 'spin up' job, to make sure it's triggered _afterwards_

```yaml
# .github/workflows/your-workflow-name.yaml
<...>
  do-things-on-runner:
    needs: spin-up-droplet # <- The ID of the 'spin up' job
    runs-on: self-hosted
    steps:
        <...>
```

Once all of the actions you want to run are completed, make sure you tear down the Droplet!

```yaml
<...>
  tear-down-droplet:
    needs: 
    - do-things-on-runner # <- The ID of previous jobs
    name: Spin Down Droplet
    if: always() # <- REQUIRED
    uses: oxide-one/workflow-runner-maker/.github/workflows/teardown.yaml@main
    with:
      name: gha-${{ github.run_id }}-${{ github.run_number }}
    secrets:
      access-token: ${{ secrets.YOUR_GITHUB_PERSONAL_ACCESS_TOKEN }}
      do-access-token: ${{ secrets.YOUR_DIGITALOCEAN_ACCESS_TOKEN }}
```

You can reference one Job ID, or multiple in your `needs:` so that they can all run on the same Droplet, one after another.

## Inputs

| Name     	| Type   	| Required 	| Default            	| Description                                     	|
|----------	|--------	|----------	|--------------------	|-------------------------------------------------	|
| `name`   	| string 	| Yes      	| None               	| Droplet name                                    	|
| `image`  	| string 	| No       	| `ubuntu-21-10-x64` 	| [Droplet Image name](https://slugs.do-api.dev/) 	|
| `region` 	| string 	| No       	| fra1               	| [Droplet Region](https://slugs.do-api.dev )     	|
| `size`   	| string 	| No       	| c-32               	| [Droplet Size](https://slugs.do-api.dev/)       	|

## Secrets

| Name              	| Description                  	|
|-------------------	|------------------------------	|
| `access-token`    	| Github Personal Access Token 	|
| `do-access-token` 	| DigitalOcean Access Token    	|


