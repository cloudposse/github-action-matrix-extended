

<!-- markdownlint-disable -->
# github-action-matrix-extended <a href="https://cpco.io/homepage?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content="><img align="right" src="https://cloudposse.com/logo-300x69.svg" width="150" /></a>
<a href="https://github.com/cloudposse/github-action-matrix-extended/releases/latest"><img src="https://img.shields.io/github/release/cloudposse/github-action-matrix-extended.svg" alt="Latest Release"/></a><a href="https://slack.cloudposse.com"><img src="https://slack.cloudposse.com/badge.svg" alt="Slack Community"/></a>
<!-- markdownlint-restore -->

<!--




  ** DO NOT EDIT THIS FILE
  **
  ** This file was automatically generated by the `cloudposse/build-harness`.
  ** 1) Make all changes to `README.yaml`
  ** 2) Run `make init` (you only need to do this once)
  ** 3) Run`make readme` to rebuild this file.
  **
  ** (We maintain HUNDREDS of open source projects. This is how we maintain our sanity.)
  **





-->

GitHub Action that when used together with reusable workflows makes it easier to workaround the limit of 256 jobs in a matrix.




## Introduction

GitHub Actions matrix have [limit to 256 items](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs#using-a-matrix-strategy)
There is workaround to extend the limit with [reusable workflows](https://github.com/orgs/community/discussions/38704)
This GitHub Action outputs a JSON structure for up to 3 levels deep of nested matrixes.
In theory run 256 ^ 3 (i.e., 16 777 216) jobs per workflow run!

| Matrix max nested level | Total jobs count limit   |
|-------------------------|--------------------------|
|         1               |            256           |
|         2               |           65 536         | 
|         3               |         16 777 216       |

If `nested-matrices-count` input is `1`, the output `matrix` would be JSON formatted string with the following structure

```yaml
{
  "include": [matrix items]
}
```

If `nested-matrices-count` input is `2` output `matrix` whould be a JSON formatted string with the following structure

```yaml
{
  "include": [{
    "name": "group name",
    "items": {
      "include": [matrix items]
    } ## serialized as string
  }]
}
```

If `nested-matrices-count` input is `3` output `matrix` would be a JSON formatted string with the following structure

```yaml
{
  "include": [{
    "name": "group name",
    "items": [{
      "name": "chunk 256 range name",
      "include": [
        "items": {
          "include": [matrix items] ## serialized as string
        }
      ]
    }] ## serialized as string

    } ## serialized as string
  }]
}
```

> [!WARNING]  
> Make sure you [restrict the concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency) of your jobs to avoid DDOS'ing the GitHub Actions API, which might cause restrictions to be applied to your account.
>
>   | Matrix max nested level | First Matrix Concurrency | Second Matrix Concurrency | Third Matrix Concurrency |
>   |-------------------------|--------------------------|---------------------------|--------------------------|
>   |         1               |            x             |              -            |             -            |
>   |         2               |            1             |              x            |             -            | 
>   |         3               |            1             |              1            |             x            |
>




## Usage

The action have 3 modes depends of how many nested levels you want. 
The settings affect to reusable workflows count and usage pattern.

## 1 Level of nested matrices

`.github/workflows/matrices-1.yml`

```yaml
  name: Pull Request
  on:
    pull_request:
      branches: [ 'main' ]
      types: [opened, synchronize, reopened, closed, labeled, unlabeled]

  jobs:
    matrix-builder:
      runs-on: self-hosted
      name: Affected stacks
      outputs:
        matrix: ${{ steps.extend.outputs.matrix }}
      steps:
        - id: setup-matrix
          uses: druzsan/setup-matrix@v1
          with:
            matrix: |
              os: ubuntu-latest windows-latest macos-latest,
              python-version: 3.8 3.9 3.10
              arch: arm64 amd64

        - uses: cloudposse/github-action-matrix-extended@main
          id: extend
          with:
            matrix: ${{ steps.setup-matrix.outputs.matrix }}
            sort-by: '[.python-version, .os, .arch] | join("-")'
            group-by: '.arch'
            nested-matrices-count: '1'          

    operation:
      if: ${{ needs.matrix-builder.outputs.matrix != '{"include":[]}' }}
      needs:
        - matrix-builder
      strategy:
        max-parallel: 10
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(needs.matrix-builder.outputs.matrix) }}
      name: Do (${{ matrix.arch }})
      runs-on: self-hosted
      steps:
        - shell: bash
          run: |
            echo "Do real work - ${{ matrix.os }} - ${{ matrix.arch }} - ${{ matrix.python-version }}"
```

## 2 Level of nested matrices

`.github/workflows/matrices-1.yml`

```yaml
  name: Pull Request
  on:
    pull_request:
      branches: [ 'main' ]
      types: [opened, synchronize, reopened, closed, labeled, unlabeled]

  jobs:
    matrix-builder:
      runs-on: self-hosted
      name: Affected stacks
      outputs:
        matrix: ${{ steps.extend.outputs.matrix }}
      steps:
        - id: setup-matrix
          uses: druzsan/setup-matrix@v1
          with:
            matrix: |
              os: ubuntu-latest windows-latest macos-latest,
              python-version: 3.8 3.9 3.10
              arch: arm64 amd64

        - uses: cloudposse/github-action-matrix-extended@main
          id: extend
          with:
            sort-by: '[.python-version, .os, .arch] | join("-")'
            group-by: '.arch'
            nested-matrices-count: '1'          
            matrix: ${{ steps.setup-matrix.outputs.matrix }}

    operation:
      if: ${{ needs.matrix-builder.outputs.matrix != '{"include":[]}' }}
      uses: ./.github/workflows/matrices-2.yml
      needs:
        - matrix-builder
      strategy:
        max-parallel: 1 # This is important to avoid ddos GHA API
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(needs.matrix-builder.outputs.matrix) }}
      name: Group (${{ matrix.name }})
      with:
        items: ${{ matrix.items }}
```

`.github/workflows/matrices-2.yml`

```yaml
  name: Reusable workflow for 2 level of nested matrices
  on:
    workflow_call:
      inputs:
        items:
          description: "Items"
          required: true
          type: string

  jobs:
    operation:
      if: ${{ inputs.items != '{"include":[]}' }}
      strategy:
        max-parallel: 10
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(inputs.items) }}
      name: Do (${{ matrix.arch }})
      runs-on: self-hosted
      steps:
        - shell: bash
          run: |
            echo "Do real work - ${{ matrix.os }} - ${{ matrix.arch }} - ${{ matrix.python-version }}"
```


## 3 Level of nested matrices

`.github/workflows/matrices-1.yml`

```yaml
  name: Pull Request
  on:
    pull_request:
      branches: [ 'main' ]
      types: [opened, synchronize, reopened, closed, labeled, unlabeled]

  jobs:
    matrix-builder:
      runs-on: self-hosted
      name: Affected stacks
      outputs:
        matrix: ${{ steps.extend.outputs.matrix }}
      steps:
        - id: setup-matrix
          uses: druzsan/setup-matrix@v1
          with:
            matrix: |
              os: ubuntu-latest windows-latest macos-latest,
              python-version: 3.8 3.9 3.10
              arch: arm64 amd64

        - uses: cloudposse/github-action-matrix-extended@main
          id: query
          with:
            sort-by: '[.python-version, .os, .arch] | join("-")'
            group-by: '.arch'
            nested-matrices-count: '1'          
            matrix: ${{ steps.setup-matrix.outputs.matrix }}

    operation:
      if: ${{ needs.matrix-builder.outputs.matrix != '{"include":[]}' }}
      uses: ./.github/workflows/matrices-2.yml
      needs:
        - matrix-builder
      strategy:
        max-parallel: 1 # This is important to avoid ddos GHA API
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(needs.matrix-builder.outputs.matrix) }}
      name: Group (${{ matrix.name }})
      with:
        items: ${{ matrix.items }}
```

`.github/workflows/matrices-2.yml`

```yaml
  name: Reusable workflow for 2 level of nested matrices
  on:
    workflow_call:
      inputs:
        items:
          description: "Items"
          required: true
          type: string

  jobs:
    operation:
      if: ${{ inputs.items != '{"include":[]}' }}
      uses: ./.github/workflows/matrices-3.yml
      strategy:
        max-parallel: 1 # This is important to avoid ddos GHA API
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(inputs.items) }}
      name: Group (${{ matrix.name }})
      with:
        items: ${{ matrix.items }}
```


`.github/workflows/matrices-3.yml`

```yaml
  name: Reusable workflow for 3 level of nested matrices
  on:
    workflow_call:
      inputs:
        items:
          description: "Items"
          required: true
          type: string

  jobs:
    operation:
      if: ${{ inputs.items != '{"include":[]}' }}
      strategy:
        max-parallel: 10
        fail-fast: false # Don't fail fast to avoid locking TF State
        matrix: ${{ fromJson(inputs.items) }}
      name: Do (${{ matrix.arch }})
      runs-on: self-hosted
      steps:
        - shell: bash
          run: |
            echo "Do real work - ${{ matrix.os }} - ${{ matrix.arch }} - ${{ matrix.python-version }}"
```






<!-- markdownlint-disable -->

## Inputs

| Name | Description | Default | Required |
|------|-------------|---------|----------|
| group-by | Group by query | empty | false |
| matrix | Matrix inputs (JSON array or object which includes property passed as string or file path) | N/A | true |
| nested-matrices-count | Number of nested matrices that should be returned as the output (from 1 to 3) | 1 | false |
| sort-by | Sort by query | empty | false |


## Outputs

| Name | Description |
|------|-------------|
| matrix | A matrix suitable for extending matrix size workaround (see README) |
<!-- markdownlint-restore -->


## Related Projects

Check out these related projects.

- [Setup matrix](https://github.com/druzsan/setup-matrix) - GitHub action to create reusable dynamic job matrices for your workflows.


## References

For additional context, refer to some of these links.

- [github-action-atmos-affected-stacks](https://github.com/cloudposse/github-action-atmos-affected-stacks) - A composite workflow that runs the atmos describe affected command




## ✨ Contributing

This project is under active development, and we encourage contributions from our community.



Many thanks to our outstanding contributors:

<a href="https://github.com/cloudposse/github-action-matrix-extended/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=cloudposse/github-action-matrix-extended&max=24" />
</a>

For 🐛 bug reports & feature requests, please use the [issue tracker](https://github.com/cloudposse/github-action-matrix-extended/issues).

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.
 1. Review our [Code of Conduct](https://github.com/cloudposse/github-action-matrix-extended/?tab=coc-ov-file#code-of-conduct) and [Contributor Guidelines](https://github.com/cloudposse/.github/blob/main/CONTRIBUTING.md).
 2. **Fork** the repo on GitHub
 3. **Clone** the project to your own machine
 4. **Commit** changes to your own branch
 5. **Push** your work back up to your fork
 6. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!

### 🌎 Slack Community

Join our [Open Source Community](https://cpco.io/slack?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content=slack) on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

### 📰 Newsletter

Sign up for [our newsletter](https://cpco.io/newsletter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content=newsletter) and join 3,000+ DevOps engineers, CTOs, and founders who get insider access to the latest DevOps trends, so you can always stay in the know.
Dropped straight into your Inbox every week — and usually a 5-minute read.

### 📆 Office Hours <a href="https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content=office_hours"><img src="https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png" align="right" /></a>

[Join us every Wednesday via Zoom](https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content=office_hours) for your weekly dose of insider DevOps trends, AWS news and Terraform insights, all sourced from our SweetOps community, plus a _live Q&A_ that you can’t find anywhere else.
It's **FREE** for everyone!
## License

<a href="https://opensource.org/licenses/Apache-2.0"><img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg?style=for-the-badge" alt="License"></a>

<details>
<summary>Preamble to the Apache License, Version 2.0</summary>
<br/>
<br/>

Complete license is available in the [`LICENSE`](LICENSE) file.

```text
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```
</details>

## Trademarks

All other trademarks referenced herein are the property of their respective owners.


---
Copyright © 2017-2024 [Cloud Posse, LLC](https://cpco.io/copyright)


<a href="https://cloudposse.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/github-action-matrix-extended&utm_content=readme_footer_link"><img alt="README footer" src="https://cloudposse.com/readme/footer/img"/></a>

<img alt="Beacon" width="0" src="https://ga-beacon.cloudposse.com/UA-76589703-4/cloudposse/github-action-matrix-extended?pixel&cs=github&cm=readme&an=github-action-matrix-extended"/>
