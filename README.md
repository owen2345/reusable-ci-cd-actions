# shareable-github-workflows
This is a repository to host all shareable github workflows

## Continuous Integration
- run tests for every pull request in the application
- the ability to run backend tests: `rspec`
- the ability to run backend frontend tests (if configured): `jest`
- the ability to run backend code style (if enabled): `rubocop`
- the ability to run frontend code style (if exist): `eslint`
- the ability to verify tests coverage (if enabled): `simplecov`

## Continuous Deployment
<img src="docs/cd-manually.png" alt="Deploy to any branch" width="100%" />
    
** Note: Continuous deployment process depends on `Kubernetes_helper` gem to run the corresponding deployment [(See documentation)](https://github.com/owen2345/kubernetes_helper)      
- The ability to manually deploy any branch to any environment

### Production deployment
- Run deployment when merged changes into Master or Main branch, such as release candidate, hotfix
- Run deployment when manually triggered to "production" env

### Staging deployment
- Run deployment automatically when a release's candidate-tests have passed
- Run deployment when manually triggered to "beta" or "beta2" environment (TODO: rename `beta` into `staging`)

## Release management
<img src="docs/release-builder.png" alt="Release builder" width="100%" />    

- The ability to generate manually a new release candidate
    - Create automatically the corresponding Release PullRequest against master/main branch
    - Add automatically all changes as the PullRequest description
    - Update automatically the changelog file with all PullRequest titles between the previous release     
    ** Note: There is an issue for PRs created by workflows: The actions do not start automatically (tests), an empty commit or any commit is required to fix it (The created PR includes a message with instructions). Issue: https://github.com/peter-evans/create-pull-request/issues/48

- Once a release candidate has been merged:
    - create the corresponding git tag
    - publish the corresponding github release with the corresponding changes (similar to changelog)

- The ability to auto-generate releases when a `hot fix` has been merged
    - Automatically calculate the new version based on the last release by incrementing the last digit, sample: '0.1.0' into '0.1.1'
    - Automatically create the corresponding git tag
    - Automatically publish the corresponding github release with the corresponding changes


## Reusable Workflows
### Continuous integration
Run tests for any pull request
```yaml
name: App tests
on:
  pull_request:

jobs:
  app-tests:
    uses: reverseretail/shareable-github-workflows/.github/workflows/tests.yml@main
    with:
      run_rubocop: false
      publish_coverage: false
```
More configurations here:
- `copy_env` (boolean, default `true`)
- `copy_db_yml` (boolean, default `false`)
- `run_rubocop` (boolean, default `true`)
- `frontend_tests` (boolean, default `false`)
- `frontend_tests_cmd` (string, default: `yarn test`)
- `run_eslint` (boolean, default `false`)
- `eslint_cmd` (string, default: `yarn eslint`)
- `rspec_cmd` (string, default: `bundle exec rspec`)
- `publish_coverage` (boolean, default `true`)
- `min_coverage` (number, default `90`)

### Continuous deployment
```yaml
name: Continous Deployment
on:
  # triggered once "App tests" (tests.yml) successfully completed for any release (auto deploy to staging)
  workflow_run:
    workflows: [ "App tests" ]
    branches: [ "release/**" ]
    types:
      - completed

  # when release PR OR Hotfix was merged, then deploy to production
  push:
    branches:
      - main
      - master

  # manually deploy any branch to a specific environment
  workflow_dispatch:
    inputs:
      deploy_env:
        type: choice
        required: true
        default: 'beta'
        options:
          - beta
          - production
        description: 'Deploy environment'

jobs:
  continuoues-deployment:
    uses: reverseretail/shareable-github-workflows/.github/workflows/cd.yml@main
    secrets:
      PROD_GOOGLE_AUTH: ${{ secrets.PROD_GOOGLE_AUTH }}
      BETA_GOOGLE_AUTH: ${{ secrets.BETA_GOOGLE_AUTH }}
```

### Sample release builder
```yaml
on:
  push: # Any pull request merged into master (hotfixes, releases or direct pushes) should publish the corresponding github release + git tag
    branches:
      - main
      - master

  workflow_dispatch: # build a new release manually (create release branch pointing to master branch)
    inputs:
      version_name:
        description: 'Release version name, sample: 1.0.0'
        required: true

name: Create Release

jobs:
  release-builder:
    uses: owen2345/reusable-ci-cd-actions/.github/workflows/release_builder.yml@main
    with:
      commit_mode: true
      create_release_pr: ${{ github.event.inputs && github.event.inputs.version_name || '' }}

```
