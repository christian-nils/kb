# Azure DevOps

## Set up Python Poetry in an Azure Pipeline

Copy-paste this code in a `azure-pipeline.yaml` file, and adapt it to your environment and Artifact feed.

```yaml
trigger: # trigger the pipeline when the branch "master" is updated (ideally this should be changed to a trigger based on the library version)
  - master

pool:
  vmImage: "ubuntu-latest" # Use Ubuntu 
strategy:
  matrix:
    BuildDeployLibrary:
      python.version: "3.9" # Set up the python version. Technically you don't have to use matrix for a single version. It is usually use if you want to run the pipeline logic with different versions
      architecture: "x64"
variables:
  POETRY_CACHE_DIR: .venv # set the cache directory for Poetry (used for all the next builds, as long as poetry.lock does not change)
  PIP_CACHE_DIR: $(Pipeline.Workspace)/.pip # set the cache directory for PIP

steps:
  - checkout: self # checkout the whole git repository
    persistCredentials: true # needed because the git repository is privates

  - task: UsePythonVersion@0 # enable the right Python version
    inputs:
      versionSpec: "$(python.version)"
    displayName: "Use python $(python.version)"

  - task: Cache@2 # enable the PIP cache, as long as the python version and the agent os are the same, the cache will be reused
    inputs:
      key: 'python | "$(Agent.OS)"'
      restoreKeys: | 
        python | "$(Agent.OS)"
        python
      path: $(PIP_CACHE_DIR)
    displayName: '[Pip] setup cache'

  - script: | # install pip and install twine (use to upload the built python package to the Artifact feed)
      python -m pip install --upgrade pip
      pip install twine
    displayName: '[Pip] twine installation'

  - task: Cache@2 # set up the Poetry cache. As long as the python version, the agent OS, and the poetry.lock files are the same, the cache will be used to install the dependencies
    inputs:
      key: 'python | "$(Agent.OS)" | poetry.lock'
      restoreKeys: | 
        python | "$(Agent.OS)"
        python
      path: $(POETRY_CACHE_DIR)
    displayName: '[Poetry] setup cache'
# install Poetry, and source it. Then set the virtualenvs in the project (to use .venv for the cache directory). Install the dependencies.
  - script: |
      curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python -
      source $HOME/.poetry/env 
      poetry config virtualenvs.in-project true
      poetry install
    displayName: '[Poetry] install dependencies'

# build the library checked out from the git repository
  - script: |
      source $HOME/.poetry/env 
      poetry build
    displayName: '[Poetry] build library'

# store the credentials to upload the python wheel to the Artifact feed. Replace 'artifactFeed' by the Artifact feed that you created. See https://docs.microsoft.com/en-us/azure/devops/artifacts/concepts/feeds?view=azure-devops
# Don't forget to go to your created feed's Settings > Permissions. Click on the three dots, and then "Allow project-scoped builds". Very crappy docs about that. See https://stackoverflow.com/questions/58780741/user-lacks-permission-to-complete-this-action-you-need-to-have-addpackage
  - task: TwineAuthenticate@1
    inputs:
      artifactFeed: 'artifactFeed'
    displayName: '[Twine] authenticate'
    
# publish the built package to your artifact. replace artifactFeed by the name of the artifact you created.
  - script: "twine upload --verbose -r artifactFeed --config-file $(PYPIRC_PATH) dist/*"
    displayName: '[Twine] publish library'
```

## Create nice repository badges for private Azure DevOps git repositories

### Package version badge
1. Go to your Artifact Feed (e.g., "artifactFeed")
2. Click on the cog icon (at the right)
3. Tick the tickbox labeled "Enable package badges"
4. You will be able to find the package version badge going to the Artifact tab and click on the three dots on the package row.
5. Click on "Create badge" and then follow the instruction to generate the image link.

### Build badge
1. Go to your project settings and navigate to Pipelines > Settings
2. Disable the slider labelled "Disable anonymous access to badges"
3. Now, you badge image will be available at `https://img.shields.io/azure-devops/build/{organization}/{project-id}/{definition-id}?label={label}`.

    1. To find the project id, go to `https://dev.azure.com/{organization}/_apis/projects/{project_name}`.
    2. To find the definition id, you need to navigate to your pipeline in DevOps, look at the URL and you will see something like `..._build?definitionId=118`, where 118 is the definition id to use in your badge. You can use the parameter "key" in the badge url to change the "Build" label to whatever you want.