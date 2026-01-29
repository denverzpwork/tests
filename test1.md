# Jenkins Commit Message Tag Handling Test

## Purpose

The goal is to verify how Jenkins processes multi-line Git commit messages when using the `when { changelog ... }` condition in a declarative pipeline.

Preliminary observations show that Jenkins evaluates **only the first line of the commit message** in the pipeline changelog context. As a result, tags placed in the body of the commit message (for example, `[skip-deploy]`) are ignored.

---

## Problem Description

The following pipeline condition is intended to skip deployment when the commit message contains `[skip-deploy]`:

```groovy
when {
    allOf {
        anyOf {
            branch 'main'
            branch 'develop'
        }
        not {
            changelog '(?s).*\\[skip-deploy\\].*'
        }
    }
}
```

However, this condition does not work as expected when `[skip-deploy]` is placed on a separate line in the commit message.

### Example Commit Message

```
Merge pull request #39 from dorsetcreative/feature/KREW-240_KIWI_import
KREW-240 kiwi import
[skip-deploy]
```

### What Jenkins Actually Uses

According to the build log, Jenkins only reads the first line:

```
Commit message: "Merge pull request #39 from dorsetcreative/feature/KREW-240_KIWI_import"
```

Relevant log output:

```
> git lfs pull origin # timeout=10
Commit message: "Merge pull request #39 from dorsetcreative/feature/KREW-240_KIWI_import"
> git rev-list --no-walk bb8562fb10314ca1fdcc4658e254376157ed24c9 # timeout=10
```

As a result:

* The `changelog` condition only sees the first line.
* The `[skip-deploy]` tag in the body is ignored.
* The deploy stage is not skipped.

---

## Proposed Workaround

Instead of relying on `when { changelog ... }`, explicitly read the full commit message from Git and evaluate it manually:

```groovy
def fullMsg = sh(
  script: "git log -1 --pretty=%B",
  returnStdout: true
).trim()

if (fullMsg =~ /\[skip-deploy\]/) {
  currentBuild.result = 'NOT_BUILT'
  error("skip deploy")
}
```

This approach ensures that:

* The complete multi-line commit message is retrieved.
* Tags in the commit body are detected correctly.
* Deployment can be reliably skipped based on `[skip-deploy]`.

---

## Test Objectives

The Jenkins test should confirm:

1. `when { changelog ... }` evaluates only the first line of a multi-line commit message.
2. `git log -1 --pretty=%B` returns the full commit message, including all lines.
3. Manual parsing of the commit message correctly blocks the deploy stage when `[skip-deploy]` is present.

---

## Proposal Jenkins pipeline

```
pipeline {
    agent any

    stages {
        stage('Checkout repository') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/denverzpwork/tests.git',
                    ]]
                )
            }
        }


        stage('Show changelog message (Jenkins view)') {
            steps {
                script {
                    echo 'Changelog messages seen by Jenkins:'
                    for (cs in currentBuild.changeSets) {
                        for (entry in cs.items) {
                            echo "- ${entry.msg}"
                        }
                    }
                }
            }
        }

        stage('Show full commit message (git log)') {
            steps {
                script {
                    def fullMsg = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()

                    echo "Full commit message from git log:\n${fullMsg}"
                }
            }
        }

        stage('Compare skip-deploy detection') {
            steps {
                script {
                    def fullMsg = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()

                    def changelogText = ''
                    for (cs in currentBuild.changeSets) {
                        for (entry in cs.items) {
                            changelogText += entry.msg + '\n'
                        }
                    }

                    echo "\nDetection results:"
                    echo "Changelog contains [skip-deploy]: ${changelogText =~ /\[skip-deploy\]/}"
                    echo "Full commit message contains [skip-deploy]: ${fullMsg =~ /\[skip-deploy\]/}"

                }
            }
        }
    }
}
```
