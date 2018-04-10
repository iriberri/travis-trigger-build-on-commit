# travis-trigger-build-on-commit
Workaround to run a build on the code of a specific commit

## Description

At the moment, the "trigger build" functionality of Travis CI's API can only be used to trigger builds on the last commit of a given branch. See: https://docs.travis-ci.com/user/triggering-builds/. The feature request to trigger a build on any commit via API is currently open in Travis CI's tracker at: https://github.com/travis-ci/travis-ci/issues/8300.

This repository is an example on how to work around this limitation and create a workflow that allows to use the API to run a build on a specific commit. **Use this approach at your own risk**.

### Note
- In Travis CI, the build triggered by the API request will be associated to the last commit of our branch, and not to the real target commit that we check out internally. Therefore, one needs to check the env vars/message of the job to know which commit was built. See: https://travis-ci.org/iriberri/travis-trigger-build-on-commit/jobs/364665902, which points to commit `b4a5660`, but uses `f683b19` internally.

## Instructions

The workflow is as follows:

- We create a specific branch in our repository that we're going to use to trigger API builds from it.
  - This branch needs to include, at least, a minimal .travis.yml with our basic setup. Other project files don't need to be in this branch.
  - This extra branch will allow us to keep all the manually triggered builds related to a specific branch. This is, however, not required - one could reuse any existing branch in their repository and trigger builds from there.
  - By creating a new branch we also avoid having the status of one of our production/development branches overwritten by the API triggered-builds
- We trigger a build request on this branch via API. This build will use a special `before_install` step, where it checks out the target commit. In this request we need to pass the target branch (`$TARGET_BRANCH`) and commit SHA (`$TARGET_COMMIT_SHA` as environment variables that will be used by git commands.


### Approach 1: keep configuration in a .travis.yml file

- We add some extra `git` steps to the configuration of our special branch, so that the first commands in the build are in charge of checking the commit that we want to build:
  ```yaml
  before_install:
    - git fetch origin $TARGET_BRANCH
    - git checkout $TARGET_COMMIT_SHA
  ```

This approach can be useful if we need to run the same commands for all our manual builds (the same `install`, `deploy` and/or `script` steps). This way, we don't need to specify the same commands in all our API build requests.

#### Request example

This example defines a `script` step. The build will use the `before_install` steps that are defined in the .travis.yml of the `manual_builds` branch.

```
body='{
 "request": {
 "branch":"manual_builds",
 "message":"Trigger build for <COMMIT_SHA>";
 "config": {
   "env": {
     "matrix": ["TARGET_COMMIT_SHA=<COMMIT_SHA> TARGET_BRANCH=<TARGET_BRANCH>"]
   },
   "script": "cat test.txt"
  }
}}'

curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Travis-API-Version: 3" \
  -H "Authorization: token <TRAVIS_API_TOKEN>" \
  -d "$body" \
  https://api.travis-ci.com/repo/<USERNAME>%2F<REPO_NAME>/requests
```

**Note:** Make sure you point your requests to the correct API endpoint depending on your case: `api.travis-ci.org` or `api.travis-ci.com`.


### Approach 2: configure extra build steps through the API request

- We trigger a build request via API, including the `git fetch` and `git checkout` commands in the body of our request.
- The .travis.yml file in this branch can include basic settings such as the `language`, and the specific build steps will be configured in the request itself.

#### Request example

This example defines the `before_install` step that will checkout the commit, as well as the `script` step:

```
body='{
 "request": {
 "branch":"empty_branch",
 "message":"Trigger build for <COMMIT_SHA>";
 "config": {
   "env": {
     "matrix": ["TARGET_COMMIT_SHA=<COMMIT_SHA> TARGET_BRANCH=<TARGET_BRANCH>"]
   },
   "before_install": ["git fetch origin $TARGET_BRANCH", "git checkout $TARGET_COMMIT_SHA"],
   "script": "cat test.txt"
  }
}}'

curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Travis-API-Version: 3" \
  -H "Authorization: token <TRAVIS_API_TOKEN>" \
  -d "$body" \
  https://api.travis-ci.com/repo/<USERNAME>%2F<REPO_NAME>/requests
```

Resulting build: https://travis-ci.org/iriberri/travis-trigger-build-on-commit/jobs/364681084

**Note:** Make sure you point your requests to the correct API endpoint depending on your case: `api.travis-ci.org` or `api.travis-ci.com`.


## Build example

* Using the workflow above to trigger a build in one of the comments from `another_branch` branch: https://travis-ci.org/iriberri/travis-trigger-build-on-commit/jobs/364665902. This build prints the content of a `test.txt` file, which does not exist in `b4a5660`, but it does in `f683b19` (which is our target commit).


