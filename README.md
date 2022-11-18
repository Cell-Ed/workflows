<img src="https://celled-images.s3-us-west-1.amazonaws.com/github/banner.jpg?raw=true" alt="celled logo">

# Distributed Workflows
This repository handles the workflows for CI/CD on Cell-Ed.
 # Usage
From another repository you can call any of the workflows defined here, to do so, please choose one of the current workflows available according to your repository needs.  
#### A brief guidance to select a workflow can be:
 - Workflows starting with **lms_** are dedicated to the LMS micro-frontend
 - Workflows starting with **models_** are reserved for DB models
 - Workflows starting with **services_** are reserved for micro-services
#### Calling a workflow from your repository
Depending your trigger conditions in your repository, calling a distributed workflow will look similar to:

	name: Publish
	on:
		release:
			types: [published]
	jobs:
		publish-on-npm:
			uses: Cell-Ed/workflows/.github/workflows/models_publish.yml@main
			secrets: inherit
The important section is **jobs**, notice that the url ends with **@main** this mean we'll be using this workflow from the branch main.  
This will allow us to work on new workflows or maintain existing ones using other branches and keep main safe.
# Technical Limitations
For distributed workflows to work, this repository needs to be public: [reusable-workflows-docs](https://docs.github.com/en/actions/using-workflows/reusing-workflows#limitations)
Because of this, please avoid to include any sensitive data in here.
# Naming Conventions
Let use **_** to split the markers on a workflow name.
![workflows_naming_convention](https://celled-images.s3.us-west-1.amazonaws.com/workflows_naming_convention.png)

# Technical Guidance
This approach will allow us to scalate and maintain the infrastructure in a much easier way.
Next, a couple of technical aspects to have in mind if we need to code a new workflow:

### Custom Actions
If your workflow has to use custom actions, we want those actions to be hosted in this repository. You can create a new folder inside ./github, please name it using camelCase like the others and make sure the name is descriptive enough for everyone to understand what this action does, others may want to use it after! 
### Checkouts
If your workflow uses custom actions you will have to perform a double checkout on your workflow to have access to all you need. Here is an example that includes some conventions for us to use in all our workflows. 
This code block can be copy/paste in almost every workflow. This way we'll always know how to read any workflow.

    jobs:
		publish-package:
			runs-on: ubuntu-latest
			defaults:
				run:
					working-directory: caller_repository
			steps:
			  - name: Checkout Caller Repository
				uses: actions/checkout@v3
				with:
				path: caller_repository
				
			  - name: Checkout Workflows Repository
				uses: actions/checkout@v3
				with:
					repository: Cell-Ed/workflows
					path: workflows_repository
The **caller_repository** folder will be created after the checkout and it will contain the branch code from the repository triggering the CI/CD (the caller)
The **workflows_repository** folder will be created after the checkout and it will contain the main branch code of this repository, allowing us to consume our custom actions.
This block code also sets the **default directory** on the **caller_repository** folder. This means that all the steps will run on that folder by default (quite convenient). 
# Contributed
 Pushing changes or creating a new workflow should be done with a PR  to main. 
If your changes are complex, please consider creating a branch for your work and make a PR to staging. After your PR is merged on staging, you can test it from any project you choose by changing the branch on the url this way:

    publish-on-npm:
			uses: Cell-Ed/workflows/.github/workflows/models_publish.yml@staging
Please notice that this repository has a major impact in all of our infrastructure, even minor changes should be take seriously.