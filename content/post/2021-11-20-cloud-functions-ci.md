---
title: "Experimenting with CI for Google Cloud Functions Monorepo"
slug: ci-cloudfunctions-monorepo
date: 2021-11-29
draft: false 
categories:
    - GCP
---

# Sensible CI for a Google Cloud Function monorepo

For the past few months I've been working for a client that has been around for long enough
to have amassed a fair bit of technical debt. After observing for quite some time team dynamics I believe
the main root causes can be summarised as:

* High employee turnover.
* Non tecnical Product Managers/Owners.
* Lack of documentation on multiple levels (both technical and business).
* Layer of custom built tools and abstractions on top of GCP services.
* Focus on quantity over quality (no time to stop reflecting on tooling improvement).

Without dwelling too much on organizational theory it's quite clear how the combination of the points above
can negatively affect the quality tooling and development experience in general. What has become particularly 
frustrating (hence the inspiration for this post) is that Jenkins builds take up to 9 hours (!!!) to propagate 
infrastructural changes to production.

I am not a software engineer by trade and I am not particularly sure how typical such scenario is in large enterprises. 
I just feel that, compared to the idylliac articles describiing DevOps best practices, wait times are way too 
long by any standard. This is obviously hampering the ability to work in an iterative manner and throw different
approaches to a problem. It is common to go for a *good enough* solution to avoid deployment times of a potentially
better approach.

## Context and Goal

Data infrastructure for the data platform (GCS buckets, Pub/Sub topics, etc.) is set up up in a declarative way
through .yaml files that get parsed by deployment scripts running as Jenkins builds. In a nutshell, every time a
commit makes it to `main`, the script parses and interacts with the **the entire** infrastructure.

Now, I am a big fan of Google Cloud Functions and I consider them a good fit for lots of data ingestion use cases.
They are particularly ideal in in scenarios where the workload doesn't justify a full-blown Dataflow pipeline, such
as:

* Pinging an API endpoint on a schedule and drop results in a BigQuery table or GCS bucket.
* Produce or consume Pub/Sub messages based on certain events.
* Retrieve model predictions from a model deployed in Google AI Platform.

### Goal

This example should be considered as an exercise (there are probably better suited approaches for production environmnets).
Its goals can be summarised in the following points:

* Set up CI (using Github Actions and Google Cloud Build) to deploy Cloud Functions to our GCP project any time
a change is deployed to `main`.
* Add logic to make sure `gcloud builds submit` is launched **only** for changed functions.
* Implement CI to run unit tests for **all** Cloud Functions everytime a change is committed. This is more of a safety 
measure (we could probably test only functions that have actually changed), but given test suites tend to be fairly 
ligthweight for cloud functions I see no harm in doing that.
* Make sure the CI pipeline can handle different runtimes (e.g Golang and Python based functions will live in different
subfolders).

## Project structure

Most of the times, Cloud Functions are so specific in scope (and implementation) that I feel they don't deserve a repo
by themselves as this would probably create a proliferation of many small repositories. The way I usually organise
project is to group functions by scope (e.g data-ingestion) or project.

```
.
├── func_1
│   ├── cloudbuild.yaml
│   ├── main.py
│   ├── requirements.txt
│   └── test_func_1.py
├── func_2
│   ├── cloudbuild.yaml
│   ├── main.py
│   ├── requirements.txt
│   └── test_func_2.py
└── func_4
    ├── cloudbuild.yaml
    ├── go.mod
    ├── main.go
    └── main_test.go
```

I've found this folder structure to be handy for a few different reasons:

1. Each function is logically separated in subfolders (`func_a`, `func_b`, etc.).
2. By having a `cloudbuild.yaml` file in each subfolder, it is easy to accomodate specific needs using deployment
[arguments](https://cloud.google.com/sdk/gcloud/reference/functions/deploy).
3. Clean separation of dependencies.
4. I can mix runtimes if I want to write something in golang or nodejs.

## Test everyting, deploy what you need

Given we're mixing up runtimes in different subfolders, we'll need to traverse through all folders and check for file
extensions before invoking the correct test suite (e.g `gotest` or `pytest`). This ha been achieved with the following
shell script:

```sh
#!/bin/sh

# For every file in repo
for i in $(ls); do
  # Check if it's a folder
  if [[ -d "$i" ]]; then
    cd $i
      is_py=0
      for f in $(ls); do
        # Check if it contains python files
        if [[ $f == *.py ]]; then
          is_py=$((is_py+1))
        fi
      done
      # Run tests if it does
      if ((is_py == 0)); then
        cd .. 
        continue
      else
        pip install -r requirements.txt
        pytest
        cd ..
      fi
  fi
done
```

This script will be executed through the following Github Action workflow file on every PR raised 
against target branches: 

```yaml
on:
  pull_request:
    branches: [main, master, dev]

name: pytest

jobs:
  deploy_functions:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: install_pytest
        shell: bash
        run: |
          pip install pytest

      - name: pytest
        shell: bash
        run: |
          chmod +x "${GITHUB_WORKSPACE}/.github/pytest_run.sh"
          source ${GITHUB_WORKSPACE}/.github/pytest_run.sh

```

### Deployment

Deployment will be achieved by using Cloud Build: each subfolder will contain a `cloudbuild.yaml` file that will 
define how the function should be deployed. This file will contain all the function-specific deployment parameters
(e.g runtime, invocation method, etc.). To be extra cautious, we will re-run tests before deploying:

```yaml
steps:
  - name: "docker.io/library/python:3.7"
    id: Test
    entrypoint: /bin/sh
    args:
      - -c
      - "pip install -r requirements.txt && pip install pytest && pytest"

  - name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
    args:
      [
        "gcloud",
        "functions",
        "deploy",
        "func_1",
        "--entry-point=hello",
        "--region=us-central1",
        "--source=.",
        "--trigger-http",
        "--runtime=python39",
      ]
```

In order to deploy **only** the functions that have been changed between the current and last revision, we'll need
to:

1. Authenticate `gcloud`. This can be done through [this](https://github.com/google-github-actions/setup-gcloud) action.
We'll also need to pass GCP auth credentials in JSON format as a repository secret.
2. Checkout the repository.
3. Get a `git diff` between the current and previous revision and extract folder names.
4. Traverse through all the folders containing changed files and run `gcloud builds submit`.

```yaml
on:
  push:
    branches: [main, master]

name: deploy

jobs:
  deploy_functions:
    runs-on: ubuntu-latest

    steps:
      - uses: google-github-actions/setup-gcloud@master
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}
          service_account_key: ${{ secrets.GCP_SA_KEY }}
          export_default_credentials: true

      - uses: actions/checkout@v2
        with:
          fetch-depth: 2

      - name: Get git diff
        id: git_diff
        shell: bash
        run: |
          echo ::set-output name=diff::$(git diff --name-only --diff-filter=AMDR @~..@ | grep "/" | cut -d"/" -f1 | uniq)

      - name: Run Cloud Build
        shell: bash
        run: |
          for i in ${{ steps.git_diff.outputs.diff }}; do
            # Don't run for CI foldere
            if [[ "$i" == ".github" ]]; then
              continue
            fi
            cd $i
                gcloud builds submit
            cd ..
          done
```

## Limitations

This approach is working just fine for my current needs. The main limitation I see is aroud mismatches between 
the python runtime version used by `ubuntu-latest` and the one used by a given function. I am not sure how likely
this might manifest into an actual problem but it might be good to match both runtimes.

## References

* This post is based on adapting this [SO answer](https://stackoverflow.com/a/60562979/9046275) to Github 
Actions. The answer explains a very similar approach on Google Cloud Build.
