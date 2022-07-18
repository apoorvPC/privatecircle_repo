# git_workflows

1. Setup a basic workflow to test changeset. When i raise a PR, a job should run to test (echo test)
2. Setup a workflow to test and deploy a change set (conditional workflow, deploy only on merge to main branch)
3. Setup a workflow to test, lint, and deploy a change set (one or more jobs based on event)
4. Setup a workflow dispatch to boot Linode instances with metadata specified input
5. Setup a workflow based on publish event
6. Setup a workflow call to link workflows 
7. Understand reusable workflows and setup workflow call which can reuse workflows from another repo


# Testing Changeset with PR 
1. Created workflow PR_test.yml
```
name: PR_Test
on:

  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v3
      - run: echo "Insert tests here"
```      
 On pushing from test_branch to main, pull req is created 
 
 Workflow triggered, checks passed
 
 <img width="820" alt="image" src="https://user-images.githubusercontent.com/53597532/178982850-31505290-c0d0-44cf-b274-c5b8d7656a89.png">
 
 # 2. Testing and Deploying Changeset 
 
 As conditional workflows based on trigger, merging a Pull Request essentially is a push to 'main'
 
 ```
 name: PR_Merge

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch

  push:
    branches: [ "main" ]


# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Changeset deployed"
 ```
 
 Now the PR triggers 2 workflows:
 
 1. Workflow for PR_test - Earlier workflow
 <img width="777" alt="image" src="https://user-images.githubusercontent.com/53597532/178986992-8ed32c36-82f7-486d-a401-6a132e800b11.png">
 
 2. Workflow for Merge_PR - triggered after merging Pull_request
 <img width="671" alt="image" src="https://user-images.githubusercontent.com/53597532/178987484-49672846-79ff-4e97-9763-89d713121257.png">

<img width="891" alt="image" src="https://user-images.githubusercontent.com/53597532/178987651-be7da578-916a-4748-b828-e0869569eaa2.png">


We're echoing "Changeset deployed" for when PR merged
 
