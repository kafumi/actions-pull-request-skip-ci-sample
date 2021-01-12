# GitHub Actions Skip CI samples

Samples of GitHub Actions workflow which skip workflow execution if commit message contains `[skip ci]`.

## `push` trigger

`github.event.head_commit.message` property in [`github` context](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#github-context) holds the commit message if workflow is triggered by `push` event. We can test if the message has `[skip ci]` keywork with GitHub Actions built-in function [`contains`](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contains). Prevent `skip_ci` job from running by setting the test result to job's [`if`](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idif) conditional. If `skip_ci` job is cancelled, the other jobs which depending on `skip_ci` job are also cancelled. 


```yaml
on: push
jobs:
  accept_commit:
    runs-on: ubuntu-latest
    if: "! contains(github.event.head_commit.message, '[skip ci]')"
    steps:
    - run: echo "build is NOT skipped"
  build:
    runs-on: ubuntu-latest
    needs: accept_commit
    steps:
```

## `pull_request` trigger

Any property in [`github` context](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#github-context) doesn't represent the commit message if workflow is triggered by `pull_request` event. We need to obtain commit message by using `git log` command with the commit hash represented by `github.event.pull_request.head.sha`. Test the commit message with `grep` command and set the result to job's `skip_ci` output. Other jobs can depend on `skip_ci` job and refer its `skip_ci` output at [`if`](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idif) conditional to cancel job execution if `skip_ci` is `true`.

```yaml
on: pull_request
jobs:
  check_commit:
    runs-on: ubuntu-latest
    env:
      SKIP_CI_KEYWORD: "[skip ci]"
    outputs:
      skip_ci: ${{ steps.check_message.outputs.skip_ci }}
    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 2
    - name: Check commit message
      id: check_message
      run: |
        message=$(git log --format=%B -n 1 ${{ github.event.pull_request.head.sha }})
        if echo "$message" | grep -q -F "${{ env.SKIP_CI_KEYWORD }}"; then
          echo "::set-output name=skip_ci::true"
        else
          echo "::set-output name=skip_ci::false"
        fi
  build:
    runs-on: ubuntu-latest
    needs: check_commit
    if: ${{ needs.check_commit.outputs.skip_ci != 'true' }}
    steps:
```