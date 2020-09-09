---
layout: classic-docs
title: "パイプライン変数"
short-title: "パイプライン変数"
description: "パイプラインの変数、パラメーター、値についての詳細情報"
categories:
  - getting-started
order: 1
version:
  - Cloud
---

パイプライン変数を使用すると、再利用可能なパイプラインを構成できます。 パイプライン変数を使用するには、[パイプライン]({{ site.baseurl }}/2.0/build-processing)を有効化し、設定ファイルで[バージョン]({{ site.baseurl }}/2.0/configuration-reference/#version) `2.1` 以上を指定する必要があります。

パイプライン変数には、次の 2 つの種類があります。

* **パイプライン値**: 設定ファイル全体で使用できるメタデータです。
* **パイプライン パラメーター**: 型指定された変数です。設定ファイルのトップ レベルに `parameters` キーで宣言します。 `parameters` は、API からパイプラインの新規実行をトリガーする際にパイプラインに渡すことができます。

## パイプライン値

パイプライン値は、あらゆるパイプライン構成で使用可能であり、事前に宣言することなく使用できます。

| 値                          | 説明                                                                                                                                          |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| pipeline.id                | A [globally unique id](https://en.wikipedia.org/wiki/Universally_unique_identifier) representing for the pipeline                           |
| pipeline.number            | パイプラインを表す、プロジェクトで一意の整数の ID                                                                                                                  |
| pipeline.project.git_url   | The URL where the current project is hosted. E.g. https://github.com/circleci/circleci-docs                                                 |
| pipeline.project.type      | The lower-case name of the VCS provider, E.g. “github”, “bitbucket”.                                                                        |
| pipeline.git.tag           | The name of the git tag that was pushed to trigger the pipeline. If the pipeline was not triggered by a tag, then this is the empty string. |
| pipeline.git.branch        | The name of the git branch that was pushed to trigger the pipeline.                                                                         |
| pipeline.git.revision      | The long (40-character) git SHA that is being built.                                                                                        |
| pipeline.git.base_revision | 以前の git リビジョン                                                                                                                               |
{: class="table table-striped"}

例

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: circleci/node:latest
    environment:
      CIRCLE_COMPARE_URL: << pipeline.project.git_url >>/compare/<< pipeline.git.base_revision >>..<<pipeline.git.revision>>
    working_directory: ~/main
    steps:
      - run: echo "This is pipeline ID << pipeline.id >>"
      - run: echo $CIRCLE_COMPARE_URL
```

## 設定ファイルにおけるパイプライン パラメーター

パイプライン パラメーターは、`.circleci/config.yml` のトップ レベルで `parameters` キーを使って宣言します。

パイプライン パラメーターは次のデータ型をサポートしています。
* 文字列
* ブール値
* 整数
* 列挙

詳しい使用方法については、「[パラメーターの構文]({{ site.baseurl }}/2.0/reusing-config/#パラメーターの構文)」を参照してください。

パイプライン パラメーターは値として参照され、スコープ `pipeline.parameters` の下で設定ファイル内の変数として使用できます。

以下の例では、2 つのパイプライン パラメーター (`image-tag`、`workingdir`) が設定ファイルの上部で定義され、後続の `build` ジョブで参照されています。

```yaml
version: 2.1
parameters:
  image-tag:
    type: string
    default: "latest"
  workingdir:
    type: string
    default: "~/main"

jobs:
  build:
    docker:
      - image: circleci/node:<< pipeline.parameters.image-tag >>
    environment:
      IMAGETAG: << pipeline.parameters.image-tag >>
    working_directory: << pipeline.parameters.workingdir >>
    steps:
      - run: echo "Image tag used was ${IMAGETAG}"
      - run: echo "$(pwd) == << pipeline.parameters.workingdir >>"
```

### API からパイプラインをトリガーするときにパラメーターを渡す

[パイプラインをトリガーする](https://circleci.com/docs/api/v2/#trigger-a-new-pipeline) API v2 エンドポイントを使用すると、特定のパラメーターの値でパイプラインをトリガーすることができます。 これを実行するには、`POST` 本体の JSON パケット内で `parameters` キーを渡します。

**Note:** Please note that the `parameters` key passed in this `POST` request is **NOT** secret.

The example below triggers a pipeline with the parameters described in the above config example (NOTE: To pass a parameter when triggering a pipeline via the API the parameter must be declared in the configuration file.).

```
curl -u ${CIRCLECI_TOKEN}: -X POST --header "Content-Type: application/json" -d '{
  "parameters": {
    "workingdir": "./myspecialdir",
    "image-tag": "4.8.2"
  }
}' https://circleci.com/api/v2/project/:project_slug/pipeline
```

## パイプライン パラメーターのスコープ

Pipeline parameters can only be resolved in the `.circleci/config.yml` file in which they are declared. Pipeline parameters are not available in orbs, including orbs declared locally in your config.yml file. This was done because access to the pipeline scope in orbs would break encapsulation and create a hard dependency between the orb and the calling config, potentially jeopardizing determinism and creating a surface area of vulnerability.


## 構成プロセスの段階とパラメーターのスコープ

### プロセスの段階

Configuration processing happens in the following stages:

- パイプライン パラメーターが解決され、型チェックされる
- パイプライン パラメーターが Orb ステートメントに置き換えられる
- Orb がインポートされる

The remaining configuration is processed, element parameters are resolved, type-checked, and substituted.

## 要素パラメーターのスコープ

Element parameters use lexical scoping, so parameters are in scope within the element they are defined in, e.g. a job, a command, or an executor. If an element with parameters calls another element with parameters, like in the example below, the inner element does not inherit the scope of the calling element.

```yaml
version: 2.1

commands:
  print:
    parameters:
      message:
        type: string
    steps:
      - run: echo << parameters.message >>

jobs:
  cat-file:
    parameters:
      file:
        type: string
    steps:
      - print:
          message: Printing << parameters.file >>
      - run: cat << parameters.file >>

workflows:
  my-workflow:
    jobs:
      - cat-file:
          file: test.txt
```

Even though the `print` command is called from the cat-file job, the file parameter would not be in scope inside the print. This ensures that all parameters are always bound to a valid value, and the set of available parameters is always known.

## パイプライン値のスコープ

Pipeline values, the pipeline-wide values that are provided by CircleCI (e.g. `<< pipeline.number >>`) are always in scope.

### パイプライン パラメーターのスコープ

Pipeline parameters which are defined in configuration are always in scope, with two exceptions:

- パイプライン パラメーターは、他のパイプライン パラメーターの定義の範囲内では有効でないため、相互に依存させることはできません。
- データ漏えいを防ぐために、パイプライン パラメーターは Orbs 本体、Orbs のインラインの範囲内では有効ではありません。

## 条件付きワークフロー

Use the `when` clause (or the inverse clause `unless`) under a workflow declaration, with a truthy or falsy value, to decide whether or not to run that workflow. Truthy/falsy values can be booleans, numbers, and strings. Falsy would be any of: false, 0, empty string, null, and NaN. Everything else would be truthy.

The most common use of this construct is to use a pipeline parameter as the value, allowing an API trigger to pass that parameter to determine which workflows to run.

Below is an example configuration using the pipeline parameter `run_integration_tests` to drive whether the workflow `integration_tests` will run.

```yaml
version: 2.1

parameters:
  run_integration_tests:
    type: boolean
    default: false

workflows:
  version: 2
  integration_tests:
    when: << pipeline.parameters.run_integration_tests >>
    jobs:
      - mytestjob

jobs:
  mytestjob:
    steps:
      - checkout
      - ... # job steps
```

The example shown above prevents the workflow `integration_tests` from being triggered unless it is explicitly invoked when the pipeline is triggered with the following in the `POST` body:

```json
{
    "parameters": {
        "run_integration_tests": true
    }
}
```

The `when` key accepts any truthy or falsy value, not just pipeline parameters, though pipeline parameters will be the primary use of this feature until more are implemented. `when` also has an inverse clause `unless`, which inverts truthiness of the condition.
