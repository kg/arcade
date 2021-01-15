# Master to main renaming guide

This is a detailed guide on how to approach renaming of the `master` branch to `main`.
It is primarily intended for repositories in the dotnet organization.
The whole process should take approximately 1-2 hours, depending on the time your PR builds take.
In case of any problems, please reach out to **@dotnet/dnceng**.
You can also use the [**First Responder** channel](https://teams.microsoft.com/l/channel/19%3aafba3d1545dd45d7b79f34c1821f6055%40thread.skype/First%2520Responders?groupId=4d73664c-9f2f-450d-82a5-c2f02756606d&tenantId=72f988bf-86f1-41af-91ab-2d7cd011db47).

Please sit back and enjoy the moment of your career where you are actually asked to delete the master branch of your project.


# Prerequisites

These prerequisites are required for a successful migration. If you're not sure about any of these, please reach out to **@dotnet/dnceng**.

Please verify that you:
- Know whether your repo is part of the [Maestro/darc dependency flow](https://github.com/dotnet/arcade/blob/master/Documentation/DependencyFlowOnboarding.md)
  - If so, have the [`darc`](https://github.com/dotnet/arcade/blob/master/Documentation/Darc.md) command installed and authenticated
  - Have PowerShell installed so that you can run scripts provided by us
- Know whether your repository is mirrored to the [internal AzDO dnceng project](https://dev.azure.com/dnceng/internal/_git)
  - Make sure you have sufficient permissions to manage branches/branch policies for the internal AzDO mirror of your repository
    - You need to have the `Force push` permission in branch security settings for the `master` branch to be able to delete it
    - Make sure you see the `Set as default branch` dropdown menu item in branch management
  - Make sure you have permissions to manage pipeline's settings in the AzDO portal
- Have permissions to manage branches and branch policies in GitHub for your repo
- Are aware of any custom hard-coded references to the `master` branch inside of your repository
  - Non-Arcade related, only specific to your repository
  - These can be some custom build scripts, documentation, makefiles...
     > Please note that GitHub has a new feature that will try to redirect you to the default branch for certain 404s,
     > e.g. https://github.com/dotnet/efcore/blob/master/README.md will lead to the `README.md` on the default `release/5.0` branch


## Step labels

Some steps are only intended for some cases, they are labelled in the following way:
- ![AzDO mirrored](images/azdo-mirrored.png) Step is intended only for repositories that are mirrored to the [internal AzDO dnceng project](https://dev.azure.com/dnceng/internal)*
- ![Maestro enabled](images/maestro-enabled.png) Step is intended only for repositories that are part of our [dependency flow](https://github.com/dotnet/arcade/blob/master/Documentation/DependencyFlowOnboarding.md)**

> \* You can tell that your repo is being mirrored by searching the git repositories in the [internal AzDO dnceng project](https://dev.azure.com/dnceng/internal/_git). In case your repository's name on GitHub is `dotnet/foo`, there should be a git repository named `dotnet-foo`. You should then also be able to find your repo in the [subscriptions.json](https://github.com/dotnet/versions/blob/master/Maestro/subscriptions.json#L627) file on which the mirroring is based.
>
> \*\* You can tell that your repo is part of our dependency flow when your repo contains the `/eng/Version.Details.xml` and `/eng/Versions.props` files. You are then also probably getting automatic updates (PRs) by the `dotnet-maestro` bot.

# Step-by-step guide

We suggest to try to not merge any PRs during the process described below. However, the instructions are ordered in a way that should prevent it and keep a consistent state even when it happens.

All of the steps are easily revert-able, so it is not a problem to go back to `master` in case you find some problems maybe only specific to your repository that would prevent you from migrating.

## Overview of steps
1. Disable Maestro subscriptions by running a following script
2. Add `main` triggers to YAML pipelines
3. Update the the build mirroring in `subscriptions.json`
4. Create the `main` branch in the internal mirrored AzDO repository
5. Change the default branch to `main` for your GitHub repository
6. Search your repository for any references to the `main` branch specific to your repo
7. Use a `darc` script to migrate channels and subscriptions
8. Change the default branch for AzDO builds for pipelines
9. Switch the default branch of the AzDO repository to `main`
10. Delete the `master` branch of the AzDO repository
11. Remove the `master` branch triggers from your YAML pipelines

## 1. ![Maestro enabled](images/maestro-enabled.png) Generate Maestro migration scripts, review and execute a script which disables all subscriptions targeting internal and GitHub repositories.

1. Download the script [from](https://raw.githubusercontent.com/dotnet/arcade/master/scripts/disable-subscriptions-prepare-migration-script.ps1).
2. Run the script `disable-subscriptions-prepare-migration-script.ps1` with short repository name (e.g. dotnet/wpf). Script generation is a safe operation which executes only darc read operations.
3. Verify that two scripts `rename-branch-in-maestro.ps1` and `disable-subscriptions-in-maestro.ps1` are generated.
4. Review the script `disable-subscriptions-in-maestro.ps1`.
5. Execute the script `disable-subscriptions-in-maestro.ps1`.

Example:
```ps
> .\disable-subscriptions-prepare-migration-script.ps1 dotnet/aspnetcore # safe operation which only generates scripts
Generating darc scripts for repository https://dev.azure.com/dnceng/internal/_git/dotnet-aspnetcore master ==> main...
Generating darc scripts for repository https://github.com/dotnet/aspnetcore master ==> main...
Files rename-branch-in-maestro.ps1 and disable-subscriptions-in-maestro.ps1 were generated.

> cat disable-subscriptions-in-maestro.ps1 # review the script which disables all subscriptions targeting internal and GitHub repositories
...
> .\disable-subscriptions-in-maestro.ps1 # disables subscriptions in Maestro
```

## 2. Add triggers for the `main` branch to all pipeline definitions. Merge this change to the `master` branch of the GitHub repo

1. Find all YAML definitions of pipelines in your repository that are triggered by changes in the `master` branch and add the `main` branch
2. Then merge this change to your `master`

**Example:**

*azure-pipelines.yml*
```yaml
# CI and PR triggers
trigger:
  batch: true
  branches:
    include:
    - master

pr:
  branches:
    include:
    - master
```

changes to

```yaml
# CI and PR triggers
trigger:
  batch: true
  branches:
    include:
    - master
    - main

pr:
  branches:
    include:
    - master
    - main
```

## 3. ![AzDO mirrored](images/azdo-mirrored.png) Fork the [versions repo](https://github.com/dotnet/versions) and open a PR to change the build mirroring in [subscriptions.json](https://github.com/dotnet/versions/blob/master/Maestro/subscriptions.json#L627)

1. Wait for code-mirroring of the trigger change from last step
2. Find the section labelled
    >`// Mirror github changes to dotnet Azure DevOps`
3. Find the row for you repo targeting the `master` branch (e.g. [this one](https://github.com/dotnet/versions/blob/ec8ac418546cdafbb8d3fbf5079e923aead33bd6/Maestro/subscriptions.json#L787))
4. Change it to `main`, e.g.
    > `"https://github.com/dotnet/xharness/blob/master/**/*",`

   to

    > `"https://github.com/dotnet/xharness/blob/main/**/*",`
4. Open a PR and get it merged (contact **@dnceng**). This will effectively disable code mirroring.


## 4. ![AzDO mirrored](images/azdo-mirrored.png) Create the `main` branch in the internally mirrored AzDO repository

1. Go to the [internally mirrored repository](https://dev.azure.com/dnceng/internal/_git) - repository should have the same name, only replace `/` with `-`, e.g. `dotnet/xharness` becomes `dotnet-xharness`
2. Wait for the code-mirror build to propagate the change from the previous step to the internal mirrored repository
    > Note: Go to the [code-mirror build](https://dev.azure.com/dnceng/internal/_build?definitionId=16&_a=summary) and filter the pipeline runs by Tags (select your repo).
3. Go to `Branches`
4. Create a new branch called `main` off of the `master` branch
5. Set the branch policies same way as it is done for `master`


## 5. Change the default branch to `main` for your GitHub repository

> Notes:
> * The `main` branch will be created as part of this step automatically. It shouldn't exist yet at this stage.
> * When user opens GitHub repo it automatically shows steps how to update local repository.
> * Automation updates target branch in all PRs.
> * GitHub raw links are automatically redirected. For example link https://raw.githubusercontent.com/dotnet/xharness/master/README.md still works even after rename and is equivalent to link https://raw.githubusercontent.com/dotnet/xharness/main/README.md.


> **Warning:** The `master` branch will be deleted during this step!

1. Navigate to your repository: `https://github.com/dotnet/[REPO NAME]`
2. In case you don't see settings tab, you don't have sufficient permissions and won't be able to proceed (please check the [prerequisites](#prerequisites))
3. Navigate to `Settings > Branches`
4. Ensure that UI looks the same as on the picture bellow. If it doesn't look the same than the GitHub rename tool isn't enabled for you. You should stop here and contact us.
5. Under the first section `Default branch` you should see branch master
6. Click `Rename branch` button to change the default branch to `main`

![Changing the default branch in GitHub](images/github-branch-rename-tool.png)


## 6. Search your repository for any references to the `master` branch specific to your repo, replace them to `main` and push them to `main`

- These can be some custom build scripts, documentation, makefiles...
- We leave you here to your own device as this can vary between repos
- Do not rename anything inside of the `/eng/common` directory


## 7. ![Maestro enabled](images/maestro-enabled.png) Use [darc](https://github.com/dotnet/arcade/blob/master/Documentation/Darc.md) to migrate default channels and subscriptions

1. Review the script `rename-branch-in-maestro.ps1` generated in step 1.
2. Execute the script `rename-branch-in-maestro.ps1`

Example:
```ps
> cat rename-branch-in-maestro.ps1  # review the script which renames branches in Maestro both for internal and GitHub repositories
...
> .\rename-branch-in-maestro.ps1 # execute rename script
...
```


## 8. ![AzDO mirrored](images/azdo-mirrored.png) Change the default branch for AzDO pipelines

- Do this for [public](https://dev.azure.com/dnceng/public) and [internal](https://dev.azure.com/dnceng/internal) projects
- Do this for all pipelines that are based off a YAML in the repo that you are working with
- You can use [this script](https://raw.githubusercontent.com/dotnet/arcade/763e8754c7e7a4b37ad76974a15dfe0ede876004/scripts/list-repo-pipelines.ps1) to list all pipelines associated with a given repo:
  > ```ps
  > .\list-repo-pipelines.ps1 -GitHubRepository "[Your repo in the form dotnet/runtime]" -PAT "[personal access token]"
  > ```

1. Go to AzDO pipelines, find your pipeline
2. Click `Edit`
   ![Edit pipeline](images/edit-pipeline.png)
3. Click the `...` three dots and then `Triggers`
   ![Piepline triggers](images/pipeline-triggers.png)
4. Click `YAML`, `Get sources` and change the `Default branch for manual and scheduled builds`
   ![Piepline triggers](images/pipeline-default-branch.png)


## 9. ![AzDO mirrored](images/azdo-mirrored.png) Go to the [internal AzDO dnceng](https://dev.azure.com/dnceng/internal/_git) mirror of your repository and switch the default branch to `main`

![Changing the default branch](images/azdo-default-branch.png)


## 10. ![AzDO mirrored](images/azdo-mirrored.png) Delete the `master` branch of the mirrored internal repository in AzDO

- For this you need to have the `Force push` permission in branch security settings


## 11. Go to the GitHub repo and remove the `master` branch triggers from your pipelines. Merge this change to the `main` branch of the GitHub repo

Similarly to the [step 2.](#2-go-to-github-and-change-yaml-for-your-pipelines-add-triggers-to-main-branch-merge-this-change-to-the-master-branch-of-the-github-repo), where you added the triggers, now remove the `master` one.

**Example:**

*azure-pipelines.yml*
```yaml
# CI and PR triggers
trigger:
  batch: true
  branches:
    include:
    - master

pr:
  branches:
    include:
    - master
```

changes to

```yaml
# CI and PR triggers
trigger:
  batch: true
  branches:
    include:
    - main

pr:
  branches:
    include:
    - main
```

This should trigger the [code-mirror build](https://dev.azure.com/dnceng/internal/_build?definitionId=16&_a=summary) which should trigger the internal pipeline.

# FAQ

### What happens to open PRs?
When branch is renamed in GitHub then open PRs against master are automatically re-targeted (forks as well).

### What if PRs get merged/opened during the process?
We suggest to try to not merge any PRs during the process described below. However, the instructions are ordered in a way that should prevent it and keep a consistent state even if it happens.

### Can I revert if something goes sideways?
All of the steps are easily revert-able, so it is not a problem to go back to master in case you find some problems maybe only specific to your repository that would prevent you from migrating.

### How do I migrate Maestro subscriptions?
Scripts for Maestro migration are part of this renaming guide.

### How will repo users learn about this?
GitHub users are automatically notified through UI that the branch was renamed and it gives them steps how to update their local repository.

![GitHub UI notification](images/github-ui-notification.png)

### What happens to links to files in my repo?
GitHub links are automatically redirected. For example https://github.com/dotnet/xharness/blob/master/README.md will still work after the rename and will point to https://github.com/dotnet/xharness/blob/main/README.md.

GitHub raw links are automatically redirected. For example link https://raw.githubusercontent.com/dotnet/xharness/master/README.md still works even after rename and is equivalent to link https://raw.githubusercontent.com/dotnet/xharness/main/README.md.

### How to revert Maestro migration?
Two update scripts are generated. There are 3 scenarios:
1. In case the `disable-subscriptions-in-maestro.ps1` script was executed **only**, to roll back, edit this script and replace argument `-d` with `-e`. For example update content of `disable-subscriptions-in-maestro.ps1` from:

```
$ErrorActionPreference = 'Stop'
# Disable targeting subscriptions for https://dev.azure.com/dnceng/internal/_git/dotnet-runtime (master)
# --------------------------------
# Disable targeting subscriptions for https://github.com/dotnet/runtime (master)
# ----------------------------------
darc subscription-status --id "032d107a-6f5d-4df8-c8c4-08d75d523d5f" -d -q
```

to:
```
$ErrorActionPreference = 'Stop'
# Disable targeting subscriptions for https://dev.azure.com/dnceng/internal/_git/dotnet-runtime (master)
# --------------------------------
# Disable targeting subscriptions for https://github.com/dotnet/runtime (master)
# ----------------------------------
darc subscription-status --id "032d107a-6f5d-4df8-c8c4-08d75d523d5f" -e -q
```
and execute `disable-subscriptions-in-maestro.ps1`.

2. When both scripts were executed, you need to generate rollback scripts using the same script which was used to generate migration scripts:
    `./disable-subscriptions-prepare-migration-script.ps1 [repo name] master main`.
    Then execute generated update script `./rename-branch-in-maestro.ps1` and all changes will be reverted.

3. Reach out to us in case of any questions or issues with these scripts.