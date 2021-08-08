[![Build Status](https://travis-ci.com/falk-werner/travis-deploy.svg?branch=main)](https://travis-ci.com/falk-werner/travis-deploy)

# travis-deploy

Example to deploy build artifacts to GitHub Releases using Travis CI.

## Motivation

[Travis CI](https://www.travis-ci.com/) allows to deploy build artifacts using GitHib Releases. This is useful to automatically add build artifacts based on a tag to the actual release. This is well described in the [Travis online documentation](https://docs.travis-ci.com/user/deployment/releases/).

However, using tags to add build artifacts to releases comes with the disadvantage, that a tag must be created before the artifacts are deployed. This makes it hard make artifacts available, which are not created by branch.

There are multiple solutions for this problem. This repository shows how to deploy artifacts not created by a tag to a special release named _unstable_.

## The _unstable_ Release

The release named _unstable_ is a special release that contains the build artifacts of the last CI build which is _not_ based on a tag.

Build artifacts of a tag build are uploaded to the regarding tag release.

_Note that the release unstable is not connected to a specific branch, any new non-tag build will overwrite the previous release._

Technically, the _unstable_ release is a branch, which is created and updated by Travis CI on demand.

## .travis.yml

To make the _unstable_ release work, some sections are added to the `.travis.yml` file.

There are also some environemt variables used, that nust be set in the travis build configuration.

| Variable      | Description                                                     |
|---------------|-----------------------------------------------------------------|
| GITHUB_APIKEY | GitHub OAuth Token to access the repository                     |
| GITHUB_USER   | Name of the user, that creates the _unstable_ release           |
| GITHUB_EMAIL  | E-Mail address of the user, that creates the _unstable_ release |

### deploy

````yaml
deploy:
  provider: releases
  api_key: $GITHUB_APIKEY
  file:
    - artifact.txt
  skip_cleanup: true
  overwrite: true
  on:
    branch: $TRAVIS_BRANCH
````

The `deploy` section is almost as described in the Travis documentation, except for the `on.branch` settings. This tweak allows to deploy artifacts from any branch.

### before_deploy

````yaml
before_deploy:
  - |
    if [[ "${TRAVIS_TAG}" = "" ]]; then
      git config --local user.name "${GITHUB_USER}"
      git config --local user.email "${GITHUB_EMAIL}"
      export TRAVIS_TAG=unstable
      curl -XDELETE --header "Authorization: token $GITHUB_APIKEY" "https://api.github.com/repos/${TRAVIS_REPO_SLUG}/git/refs/tags/${TRAVIS_TAG}" || true
      git tag -d "${TRAVIS_TAG}" || true
      git tag "${TRAVIS_TAG}"
    fi
````

The `before_deploy` section contains a script that is executed before the deploy stage.
The script is used to create the _unstable_ release on non-tag builds.
To prevent a mixup of artifacts from different builds, the _unstable_ release will be removed and re-created on each build.

_Note that the name unstable was used in favor latest, since lastes is already reserved by GitHub._

### branches

The `branches` section is used to exclude the tag _unstable_ from Travis builds. This is to avoid build c<cles since tag _unstable_ is created by a Travis build which may lead to a Travis build if not excluded.
