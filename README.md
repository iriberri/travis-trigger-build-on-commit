# travis-trigger-build-on-commit
Workaround to run a build on the code of a specific commit

## Description

### Note

## Instructions

To trigger a build on a given commit you need to perform a build request to the `manual_build` branch (via API, or via Travis' "Trigger build" feature) with the following environment variables configured:
- `$TARGET_COMMIT_SHA`: SHA of the commit that you want to build
- `$TARGET_BRANCH`: Name of the branch that contains your target commit

An example request is the following:

```
body='{
 "request": {
 "branch":"manual_builds",
 "config": {
   "env": {
     "matrix": ["TARGET_COMMIT_SHA=<COMMIT_SHA>", "TARGET_BRANCH=<TARGET_BRANCH>"]
   }
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

## Build examples



