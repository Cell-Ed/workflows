<img src="https://celled-images.s3-us-west-1.amazonaws.com/github/banner.jpg?raw=true" alt="celled logo">

# Distributed Workflows
This repository handles the workflows for CI/CD on Cell-Ed.
# Glossary
- [Distributed Workflows](#distributed-workflows)
- [Glossary](#glossary)
- [Usage](#usage)
		- [A brief guidance to select a workflow](#a-brief-guidance-to-select-a-workflow)
		- [Calling a workflow from your repository](#calling-a-workflow-from-your-repository)
- [Technical Limitations](#technical-limitations)
- [Naming Conventions](#naming-conventions)
- [Technical Guidance](#technical-guidance)
		- [Custom Actions](#custom-actions)
		- [Checkouts](#checkouts)
- [Contributing](#contributing)
# Usage
From another repository you can call any of the workflows defined here, to do so, please choose one of the current workflows available according to your repository needs.  
### A brief guidance to select a workflow
 - Workflows starting with **lms_** are dedicated to the LMS micro-frontend
 - Workflows starting with **models_** are reserved for DB models
 - Workflows starting with **services_** are reserved for micro-services
### Calling a workflow from your repository
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
### Claude Code workflows
 - **claude_code_review.yml** reviews non-draft pull requests and keeps one outcome comment on each up to date. PRs into `staging` are reviewed only when they carry the `claude-review` label (set `request_only_branches` to change which branches); PRs into any other branch are always reviewed. Drafts wait until they are ready for review; fork and Dependabot PRs are never reviewed, because their runs get no secrets.
 - **claude_code_mention.yml** answers `@claude` in issues, PR comments and PR reviews.

Both need the org secret `CLAUDE_CODE_OAUTH_TOKEN` and the Claude GitHub App.

**Secret scan (CEL-476).** `claude_code_review.yml` also has a job, `Secret scan (gitleaks)`, that runs on every pull request event of every repository that calls it, whatever the review settings: drafts, forks, Dependabot, staging PRs without the `claude-review` label, and `labeled` events too. It checks out the pull request, verifies a pinned gitleaks binary against a hard-coded SHA-256 and scans only the commits the pull request adds (`base..head`, merge commits excluded). It has `contents: read` and uses no secrets. A finding fails the job and the log shows the file, line, rule and commit with the value redacted. The repository's own `.gitleaks.toml` and `.gitleaksignore` apply, and inline `gitleaks:allow` comments are ignored: a suppression is a `.gitleaksignore` change that goes through review. A scan of the full history is tracked in CEL-477. To update gitleaks, change `GITLEAKS_VERSION` and `GITLEAKS_SHA256` together in the job (the steps are in the job's comments).

**Review styles (CEL-263).** `claude_code_review.yml` has two ways to review. Callers don't change: a repository opts in with **repository variables** (Settings → Secrets and variables → Actions → Variables), or a caller can pass the inputs of the same name.

| Variable (input) | Values | Default |
|---|---|---|
| `CLAUDE_REVIEW_STYLE` (`review_style`) | `trace`: one reviewer that traces every changed entry point from input to database and back, checks access, limits, failure paths, concurrency and data exposure, and on later pushes re-reviews against its own last review and the author's answers. `plugin`: the code-review plugin | `trace` |
| `CLAUDE_REVIEW_MODEL` (`review_model`) | a model id | `claude-sonnet-5-5` |
| `CLAUDE_REVIEW_EFFORT` (`review_effort`) | `low`, `medium`, `high`, `xhigh`, `max` | `high` |
| `CLAUDE_REVIEW_TRIAL` (`review_trial`) | `true`: also run the other style; the chosen style still submits the formal review, the other posts its own comment only and never fails the checks | `false` |

The trace review reads the pull request from files the workflow prepares (the diff, the conversation, its previous review, what changed since, the pull request's commits and each changed file's recent history), and runs with file tools only: no shell or `git` commands, no GitHub access, no Write or Edit, and no reads of the process environment, runner temp files, hidden folders in the runner's home or git config. The checkout keeps no token, and the outcome comment redacts anything shaped like a credential and then approves nothing. Trace on Sonnet at high effort is the review standard (lead, 14 Oct 2026): it blocked a change on three high-severity security findings that the plugin approved. To fall back to the plugin for one repository, set its `CLAUDE_REVIEW_STYLE` variable to `plugin`; an organization variable of the same name does it for every repository. Unknown values fall back to the defaults.

**Formal review (CEL-263).** When the org secrets `CLAUDE_REVIEW_APP_CLIENT_ID` and `CLAUDE_REVIEW_APP_PRIVATE_KEY` are set, `claude_code_review.yml` also submits the result as a pull request review from the **Cell-Ed Claude Review** GitHub App, on the reviewed head commit:

- **Approve** when Claude ends with `VERDICT: APPROVE`.
- **Request changes** when it ends with `VERDICT: REQUEST_CHANGES`, which it does for correctness, security or privacy problems, missing tests for changed behaviour, or anything else that must be fixed before merging.
- **Nothing** when the run fails, there's no verdict, or the PR head moved on during the review. This fails closed: no approval.

The app's approval counts toward the branch rule's required approvals. In the branch rule, turn on both:

- "Dismiss stale pull request approvals when new commits are pushed";
- "Require approval of the most recent reviewable push".

The second one matters: a push can land between the review's head check and its approval, so the approval arrives *after* the push but names the older commit. Dismissal doesn't catch that. Requiring approval of the latest push does, so an approval of an older commit never satisfies the rule. Without the two secrets, only the outcome comment is posted.

**Setting up the app** (org admin, once):
1. Organization settings → Developer settings → GitHub Apps → New GitHub App, named "Cell-Ed Claude Review".
2. Set any homepage URL and turn the webhook off.
3. Repository permissions: Pull requests **Read and write**, Contents **Read and write**, Metadata **Read**. Install it on **this organization only**.
   - Contents must be **write**: GitHub only counts approvals from reviewers who can push to the repository. With Contents read, the approval is posted but shows `canPush=false` and doesn't meet the rule.
   - The review job's token is still limited to `pull-requests: write`, so the job can't push.
   - Keep the private key only in the org secret: anyone holding it could mint a token that pushes.
4. Create a private key, then install the app on the repositories that use the review.
5. Add org Actions secrets `CLAUDE_REVIEW_APP_CLIENT_ID` (the app's Client ID) and `CLAUDE_REVIEW_APP_PRIVATE_KEY` (the whole `.pem`), available to those repositories.

The review runs on `pull_request` from the same repository, so someone with write access could change a repo's caller workflow in their own PR. Protect `.github/` with CODEOWNERS or a ruleset once more people have write access. The caller must grant the permissions the job uses, because a called workflow cannot hold more than its caller; the header of each file has the caller to copy.
# Technical Limitations
For distributed workflows to work, this repository needs to be public: [reusable-workflows-docs](https://docs.github.com/en/actions/using-workflows/reusing-workflows#limitations)
Because of this, please avoid to include any sensitive data in here.
# Naming Conventions
Let use **_** to split the markers on a workflow name.
![workflows_naming_convention](https://celled-images.s3.us-west-1.amazonaws.com/workflows_naming_convention.png)

# Technical Guidance
This approach will allow us to scale and maintain the infrastructure in a much easier way.
Next, a couple of technical aspects to have in mind if we need to code a new workflow:

### Custom Actions
If your workflow has to use ![custom actions](https://docs.github.com/en/actions/creating-actions), we want those actions to be hosted in this repository. You can create a new folder inside ./github, please name it using camelCase like the others and make sure the name is descriptive enough for everyone to understand what this action does, others may want to use it laterwards! 
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
The ``caller_repository`` folder will be created after the checkout and it will contain the branch code from the repository triggering the CI/CD (the caller)
The ``workflows_repository`` folder will be created after the checkout and it will contain the main branch code of this repository, allowing us to consume our custom actions.
This code block above also sets the ``default directory`` on the ``caller_repository`` folder. This means that all the steps will run on that folder by default (quite convenient). 
# Contributing
 Pushing changes or creating a new workflow should be done with a PR  to main. 
If your changes are complex, please consider creating a branch for your work and make a PR to staging. After your PR is merged on staging, you can test it from any project you choose by changing the branch on the url this way:

    publish-on-npm:
			uses: Cell-Ed/workflows/.github/workflows/models_publish.yml@staging

Please notice that this repository has a major impact in all of our infrastructure, even minor changes should be taken seriously.