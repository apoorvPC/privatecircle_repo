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
 
# 3. Conditional Workflow 

Event triggers can be multiple, but all events which trigger the workflow have to be listed.

if: condition can be used to match events or repo names etc, and run jobs based on those conditions.

```
name: Conditional Workflow Check
on:
  push:
   branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Lint
        if: ${{ github.ref == 'ref/heads/main' }} && github.event_name == 'pull_request'
        run: echo "Lint checks based on conditional flow"
      - name: Test
        run: echo "Testing workflow"
      - name: Deploy
        if: github.event_name == 'push' && ${{ github.ref == 'ref/heads/main' }}
        run: echo "Changeset deployed"
```
**Condition 1**  When Pull Request created, Lint and Test jobs will run but Deploy won't 
![image](https://user-images.githubusercontent.com/109505635/179493450-43713d80-2b33-4fcf-afbf-a2a5c3189973.png)

**Condition 2**  When PR is merged (push to main), then Deploy job will run
![image](https://user-images.githubusercontent.com/109505635/179493854-3d1b85d1-ee6f-4d51-9bfc-9579820f0eb1.png)

# 4. Workflow dispatch based on inputs

workflow dispatch and workflow call can have user inputs specified during workflow run.

We can trigger jobs based on metadata input.
Following workflow takes DiskType, Region as choice, Tags as String, and Print_tags as bool.

When the Print_tags is checked, we display the metadata specified by user.

```

name: Workflow Dispatch
on:
  workflow_dispatch:
    inputs:
      DiskType:
        description: "Select Disk Type"
        required: true
        default: "Premium SSD"
        type: choice
        options:
          - Premium SSD
          - Redundant SDD
          - Optimised HDD
        
      Region:
        description: "Select your Region"
        required: true
        type: choice
        options:
          - Central US
          - Central India
          - South India
          
      Tags:
        description: "Test scenario tags"
        required: true 
        type: string
        
      print_tags:
        description: 'True to print to STDOUT'
        required: true 
        type: boolean
        
jobs:
 build:
   runs-on: ubuntu-latest
   steps:
     - name: Display_Metadata
       if:  ${{ inputs.print_tags }} 
       run: echo  "The tags are ${{ inputs.tags }}, Region Selected ${{ inputs.region }}, DiskType ${{ inputs.DiskType }}"
```

![image](https://user-images.githubusercontent.com/109505635/179502661-0d7d815d-d322-4b5d-a6cf-f313fc3eba34.png)


This is the output of workflow:

![image](https://user-images.githubusercontent.com/109505635/179502832-8a823319-413e-4882-b0c2-e0aa11613c93.png)

# 5. Workflow triggered on "publish" (tags etc)

Created a workflow to listen to when tags are "published" , and then create a release for that tag.

git checkout <commit id>

git tag "tagName"

git push --tags

```
name: Create release on tags
on:
  push:
    tags:
      - '*'


jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Create Release
        uses: actions/create-release@latest
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
```

ERROR: Workflow not getting triggered when tag is pushed

RES: I found out that if the tag was previously created (locally) before the workflow was created, no matter how many times I deleted and re-pushed the tag, it would not trigger until I deleted the tag locally and recreated it. The action does not seem to work for tags created before the workflow.

```
name: Create release on tags
on:
  release:
    types: [published, created]


jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Create Release
        run: echo "Release created"
```
Workflow triggered when new release is created/drafted from tag.

![image](https://user-images.githubusercontent.com/109505635/179680005-cf857e34-803e-460f-a590-00ccf376113a.png)

# 6. Reusable workflows
  
  Created:
  
  1. Caller workflow - having workflow dispatch, takes user inputs on trigger and passes value to called workflow.
  
  2. Called workflow - Has jobs to display the data passed by invoking workflow.
  
  **caller-workflow.yml**
  ```
  name: Call a reusable workflow

on:
  workflow_dispatch:
    inputs:
      username:
        description: Enter username
        required: true
        type: string
      password:
        description: Enter password
        required: true
        type: string

jobs:
  call-workflow-passing-data:
    uses: apoorvPC/privatecircle_repo/.github/workflows/called-workflow.yml@main 
    with:
      username: ${{ inputs.username }}
      password: ${{ inputs.password }}
  ```
  
  **called-workflow.yml**
  ```
  name: Reusable workflow example

on:
  workflow_call:
    inputs:
      username:
        required: true
        type: string
      password:
        required: true
        type: string
        
jobs:
 build:
   runs-on: ubuntu-latest
   steps:
     - name: Display Data
       run: echo "Username is ${{ inputs.username }}, Password is ${{ inputs.password }}"
  ```
  
  ![image](https://user-images.githubusercontent.com/109505635/179735416-8fc34644-f3ed-4af5-a3c7-67c3b7daf635.png)

  ![image](https://user-images.githubusercontent.com/109505635/179735486-1decf26d-286a-4a12-95c2-acffbd4829f5.png)
  
 # 7. Reusable workflows - based on conditional statements & Passing Secrets
  
  Created two workflows (repo - apoorvPC/privatecircle_repo) to be called conditionally - based on user input.
  
  First workflow runs NPM Tests, and accepts a **secret** from caller-workflow
  ```
name: Workflow to be invoked #1
on:
  workflow_call:
    secrets:
      TOKEN:
        required: true
  
        
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      STATIC_VALUE: true # populated
      TOKEN: ${{ secrets.TOKEN }}
    steps:
      - name: npm test
        run: echo "NPM Token is ${{ secrets.TOKEN }}"
  ```
  
  Second workflow runs Sonar tests, and accepts a **secret** from caller-workflow
  ```
  
name: Workflow to be invoked #2
on:
  workflow_call:
    secrets:
      TOKEN:
        required: true
  
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      STATIC_VALUE: true # populated
      TOKEN: ${{ secrets.TOKEN }}
    steps:
      - name: Sonar test
        run: echo "Sonar Token is ${{ secrets.TOKEN }}"
  ```
  
  Finally, we have the caller-workflow in another repo (repo - apoorvPC/TestRepo), with secrets present in this same repo
  
 ```
name: Test Caller workflow

on:
  workflow_dispatch:
    inputs:
      Test:
        description: "Select Test type"
        required: true
        type: string
        

jobs:
 calling-workflow-2:
    if: ${{ inputs.Test=='npm' }}
    uses: apoorvPC/privatecircle_repo/.github/workflows/pan-repo-workflow2.yml@main
    secrets:
      TOKEN: ${{ secrets.NPM_AUTH_TOKEN }}
    
  
 calling-workflow-1:
    if: ${{ inputs.Test=='sonar' }}
    uses: apoorvPC/privatecircle_repo/.github/workflows/pan-repo-workflow1.yml@main
    secrets:
      TOKEN: ${{ secrets.SONAR_AUTH_TOKEN }}
 ```

NOTE: tOKEN IS PASSED BETWEEN WORKFLOWS BUT CANNOT BE ECHOED AS IS ENCRYPTED.

When user gives input as 'npm', NPM test workflow is called and NPM token is passed. Sonar workflow is skipped.

![image](https://user-images.githubusercontent.com/109505635/179910257-34ea0073-9761-466c-a06c-69c026457946.png)
