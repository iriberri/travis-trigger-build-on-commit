# travis-trigger-build-on-commit
Workaround to run a build on the code of a specific commit

## Description

At the moment, the "trigger build" functionality of Travis CI's API can only be used to trigger builds on the last commit of a given branch. See: https://docs.travis-ci.com/user/triggering-builds/. The feature request to trigger a build on any commit via API is currently open in Travis CI's tracker at: https://github.com/travis-ci/travis-ci/issues/8300.

This repository is an example on how to work around this limitation and create a workflow that allows to use the API to run a build on a specific commit. **Use this approach at your own risk**.

The workflow is as follows:
- We create a specific branch in our repository that we're going to use to trigger API builds from it. (In this repository, the branch is named `manual_builds`).
  - This will allow us to keep all the manually triggered builds related to a specific branch. This is, however, not required - one could reuse any existing branch in their repository and trigger builds from there.
- We add some extra steps to the configuration of our special branch, so that the first commands in the build are in charge of checking the commit that we want to build:
  ```yaml
  before_install:
    - git fetch origin $TARGET_BRANCH
    - git checkout $TARGET_COMMIT_SHA
  ```
- We trigger a build request on this branch via API, passing the target branch and commit as an environment variable that will be used by the snippet above. 

### Note
- In Travis CI, the build triggered by the API request will be associated to the last commit of our branch, and not to the real target commit that we checkout internally. Therefore, one needs to check the env vars of the job to know which commit was built. See: https://travis-ci.org/iriberri/travis-trigger-build-on-commit/jobs/364665902, which points to commit `b4a5660`, but uses `f683b19` internally.

- This is just a work around. Again: use this at your own risk.

## Instructions

To trigger a build on a given commit you need to perform a build request to the `manual_build` branch (via API, or via Travis' "Trigger build" feature) with the following environment variables configured:
- `$TARGET_COMMIT_SHA`: SHA of the commit that you want to build
- `$TARGET_BRANCH`: Name of the branch that contains your target commit

In this request you can also define the steps of your build (be careful to avoid overwriting our `before_install` section that is in charge of checking out the target commit). For example, the example below defines a new "script" section:

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

## Build example

* Using the workflow above to trigger a build in one of the comments from `another_branch` branch: https://travis-ci.org/iriberri/travis-trigger-build-on-commit/jobs/364665902. This build prints the content of a `test.txt` file, which does not exist in `b4a5660`, but it does in `f683b19` (which is our target commit).


