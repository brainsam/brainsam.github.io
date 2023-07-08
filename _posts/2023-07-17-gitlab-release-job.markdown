---
layout: post
title:  "Automated versioning and release with standard-version"
date:   2023-07-17 10:00:00 +0400
categories: devops
---

**Problem**: Development team wants to reduce time to release management and automatic version calculating in gitlab

Possible solution:

1. Adopt [conventional commits](https://conventionalcommits.org) format
2. Use [standard-version](https://github.com/conventional-changelog/standard-version) project for automating versioning. It is deprecated, but still useful and does it's work.
3. Create gitlab jobs for releases

Requirements:
* git
* [standard-version](https://github.com/conventional-changelog/standard-version)
* [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog/blob/master/packages/conventional-changelog-cli/README.md)
* [commitlint](https://commitlint.js.org/)
* gitlab ci with docker/kubernetes runners


## 1. Adopt conventionalcommits

To enforce following the commit conventions can be helpful [commitlint](https://commitlint.js.org/).

This is an example job to control if commit messages are match format:
```yaml
commitlint:
  stage: test
  image: node:18
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
  before_script:
    - npm install -g @commitlint/cli
    - |
        echo "module.exports = {extends: ['@commitlint/config-conventional']}" \
        > commitlint.config.js
    - git fetch origin ${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}
  script: commitlint --from=origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME
```
## 2. Use [standard-version](https://github.com/conventional-changelog/standard-version) project for automating versioning.

Since every develper follows commit format, we can automatically generate new versions in [semver](https://semver.org/) and create release notes.

1. Install standard-version library
    ```bash
    npm install -g standard-version
    ```
2. For nodejs projects ensure that `version` field is defined in package.json file. For other types define your project settings in .versionrc.json:
    * Gradle
        ```bash
        npm install @damlys/standard-version-updater-gradle
        ```
        ```json
        #.versionrc.json
        {
        "packageFiles": [
            {
            "filename": "build.gradle",
            "updater": "node_modules/@damlys/standard-version-updater-gradle/dist/build-gradle.js"
            }
        ],
        "bumpFiles": [
            {
            "filename": "build.gradle",
            "updater": "node_modules/@damlys/standard-version-updater-gradle/dist/build-gradle.js"
            }
        ]
        }
        ```
    * Maven:
        ```bash
        npm install standard-version-maven
        ```
        ```json
        #.versionrc.json
        {
        "packageFiles": [
            {
            "filename": "./pom.xml",
            "updater": "node_modules/standard-version-maven/index.js"
            }
        ],
        "bumpFiles": [
            {
            "filename": "./pom.xml",
            "updater": "node_modules/standard-version-maven/index.js"
            }
        ]
        }
        ```
    * Any other with version file in plain text:
        ```json
        #.versionrc.json
        {
        "packageFiles": [
            {
            "filename": "version.txt",
            "type": "plain-text"
            }
        ],
        "bumpFiles": [
            {
            "filename": "version.txt",
            "type": "plain-text"
            }
        ]
        }
        EOF
        )
        ```
3. Ensure that `repository` field is defined in ./package.json file. It user by standard-version to correctly set links to commmits in CHANGELOG.md file:
    ```json
    #package.json
    {
        "repository": "https://github.com/brainsam/braninsam.github.io"
    }
    ```
4. 🛠️ Now, we can create release:
    ```bash
    standard-version
    ```
    the tool will do:
    1. seek for previous version in tags, then in `packageFiles.filename`
    2. calculate new version
    3. generate changelog
    4. write new version to `bumpFiles.filename`
    5. commit and add tag

5. 🎉 Done, we have new release revision, and can build and publish package.

## 3. Create gitlab job

To automate release process after previous settings we can add gitlab ci job and run it instead of Merge request by hands:

```yaml
release:
  image: node:lts
  stage: release
  variables:
    GIT_DEPTH: 0
    GIT_SUBMODULE_STRATEGY: normal
  rules:
    - if: $CI_COMMIT_TAG # we have to avoid job duplication
      when: never
    - if: $CI_COMMIT_BRANCH
      when: manual
  before_script: npm install -g standard-version conventional-changelog-cli
  script:
    - git config user.name "gitlab"
    - git config user.email "noreply@example.com"
    - git fetch --tags
    - git fetch origin $CI_DEFAULT_BRANCH
    - git checkout --track origin/$CI_DEFAULT_BRANCH
    - git merge --no-edit --no-ff origin/$CI_COMMIT_BRANCH
    # deleting possible pre-release tags to correctly build release notes
    # https://github.com/conventional-changelog/standard-version/issues/203
    - |
      git tag -l \
      | grep -E "[0-9]-[A-Za-z]+\.[0-9]+" \
      | xargs git tag -d \
      || exit_code=$?
    # by default standard version creates additional commit, that we do not want to.
    # Best solution is to keep only release commits in main branch, so we
    # use postcommit lifecycle sript to merge last to commits into
    # one before pushing to origin repository.
    - |
      standard-version \
      --scripts.postcommit="git reset HEAD~1 \
        && git commit --all --amend --reuse-message=HEAD"
    - git checkout $CI_COMMIT_BRANCH && git merge --ff-only $BASE_BRANCH
    - |
      git push \
        --follow-tags \
        https://gitlab-ci-token:${PUSHER_TOKEN}@${CI_REPOSITORY_URL#*@} \
        $BASE_BRANCH \
        $CI_COMMIT_BRANCH
    # Call conventional-changelog utility to create release notes  for gitlab release
    - |
      conventional-changelog \
      --preset=angular \
      --release-count=2 \
      --outfile=RELEASE-NOTES.md
    # Adding gitlab release
    - |
      release-cli create \
      --name "${CI_PROJECT_NAME}-$(git describe --tags)" \
      --description RELEASE-NOTES.md \
      --tag-name "$(git describe --tags)"
```