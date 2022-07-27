# cloudify-action-rest

This blueprint example shows how to run a GitHub Actions workflow from Cloudify and wait until it completes. The example blueprint runs the workflows in the .github/workflows directory, which are all just simple scripts, one after the other:

* run-basic-script pipeline: Runs a [basic script](./workflow_scripts/basic_script.sh) that has no delay
* run-script-with-delay: Runs a [script with an artificial delay](./workflow_scripts/script_with_delay.sh) to demonstrate how the logic works to wait for completion
* run-script-with-inputs: Runs a single workflow command that accepts an input.

# Explanation

The "wait until workflow is completed" logic is complicated by the nature of GitHub's API. Creating a [workflow dispatch event](https://docs.github.com/en/rest/reference/actions#create-a-workflow-dispatch-event) is asynchronous, and **does not** return any information, other than an HTTP response. Therefore, the logic to run a pipeline and wait for it to finish must actually execute 4 API calls:

1. Get the ID of the previous workflow run ([see here](./templates/get-previous-run.yaml))
2. Initiate a new workflow run  ([see here](./templates/run-workflow.yaml) or [here with inputs](./templates/run-workflow-with-inputs.yaml))
3. Obtain the ID of the newly executed workflow and make sure that it is not the same as the previous run ([see here](./templates/get-run-id.yaml))
  * This avoids the race condition where a call to the list API does not yet contain the ID of the newly executed workflow from step 2 (again, this is a limitation of GitHub's API)
4. Check the status of that workflow, and wait for it to complete ([see here](./templates/get-workflow-status.yaml))

Each of these API calls is implemented as its own lifecycle operation in the [sample blueprint](./blueprint.yaml). Each API call requires its own lifecycle operation for two reasons:

1. Inputs must be passed between the steps (e.g., the previous run ID must be passed from step 1 to step 3)
2. Placing all API calls into the same operation would retry **all of the calls** on failure of one of the calls. For example: if the call to check status failed, the API call to execute the workflow would **also** be retried

# Usage

1. Create a `github_token` secret in your Cloudify manager with `repo` scope permissions
2. Upload and execute the blueprint, changing the inputs to match your repository location.

# Caveat

The workflow that you are attempting to run must have been run at least once. Otherwise, the call to retrieve a previous workflow ID (step 1) will fail with an empty list.
