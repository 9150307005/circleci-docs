---
layout: classic-docs
title: "Executor タイプを選択する"
short-title: "Executor タイプを選択する"
description: "Docker、Machine、および macOS Executor タイプの概要"
categories:
  - containerization
order: 10
version:
  - Cloud
  - Server v2.x
---

[custom-images]: {{ site.baseurl }}/2.0/custom-images/ [building-docker-images]: {{ site.baseurl }}/2.0/building-docker-images/ [server-gpu]: {{ site.baseurl }}/2.0/gpu/

以下のセクションに沿って、利用可能な Executor タイプ (`docker`、`machine`、`windows`、`macos`) について説明します。

* 目次
{:toc}

## 概要
{:.no_toc}

*Executor タイプ*は、ジョブを実行する基盤テクノロジーまたは環境を定義します。 CircleCI では、以下の 4 つの環境のいずれかでジョブを実行できます。

* Docker イメージ内 (`docker`)
* Linux 仮想マシン (VM) イメージ内 (`machine`)
* macOS VM イメージ内 (`macos`)
* Windows VM イメージ内 (`windows`)

['.circleci/config.yml']({{ site.baseurl }}/2.0/configuration-reference/) で Executor タイプと適切なイメージを指定することで、ジョブごとに異なる Executor タイプを指定することも可能です。 *イメージ*は、実行環境を作成するための指示を含むパッケージ化されたシステムです。 *コンテナ*または*仮想マシン*は、イメージの実行インスタンスを指す用語です。 たとえば以下のように、ジョブごとに Executor タイプとイメージを指定できます。

* Docker イメージ (`docker`) を必要とするジョブには、Node.js または Python のイメージを使用します。 CircleCI Docker Hub にある[ビルド済み CircleCI Docker イメージ]({{ site.baseurl }}/2.0/circleci-images/)を使用すると、Docker について完全に理解していなくてもすぐに着手できます。 このイメージはオペレーティング システムの全体ではないので、通常はソフトウェアのビルドの効率化が図れます。 
* Linux 仮想マシン (VM) の完全なイメージ (`machine`) を必要とするジョブは、Ubuntu バージョン (16.04 など) を使用します。
* macOS VM イメージ (`macos`) を必要とするジョブには、Xcode バージョン (10.0.0 など) を使用します。

## Docker の使用

The `docker` key defines Docker as the underlying technology to run your jobs using Docker Containers. Containers are an instance of the Docker Image you specify and the first image listed in your configuration is the primary container image in which all steps run. If you are new to Docker, see the [Docker Overview documentation](https://docs.docker.com/engine/docker-overview/) for concepts.

Docker increases performance by building only what is required for your application. Specify a Docker image in your [`.circleci/config.yml`]({{ site.baseurl }}/2.0/configuration-reference/) file that will generate the primary container where all steps run:

```yaml
jobs:
  build:
    docker:
      - image: buildpack-deps:trusty
```

In this example, all steps run in the container created by the first image listed under the `build` job. To make the transition easy, CircleCI maintains convenience images on Docker Hub for popular languages. See [Using Pre-Built CircleCI Docker Images]({{ site.baseurl }}/2.0/circleci-images/) for the complete list of names and tags. If you need a Docker image that installs Docker and has Git, consider using `docker:stable-git`, which is an offical [Docker image](https://hub.docker.com/_/docker/).

### Docker イメージのベスト プラクティス
{:.no_toc}

* `latest` や `1` のような可変タグを `config.yml file` でイメージのバージョンとして使用することは避けてください。 例に示すように、`redis:3.2.7`、`redis@sha256:95f0c9434f37db0a4f...` といった正確なイメージ バージョンまたはダイジェストを使用することをお勧めします。 可変タグは、多くの場合、ジョブ環境で予期しない変更を引き起こします。 CircleCI は、可変タグがイメージの最新バージョンを返すことを保証できません。 `alpine:latest` と指定しても、実際には 1 か月前の古いキャッシュが取得される場合があります。

* 実行中に追加ツールがインストールされるために実行時間が長くなる場合は、「[カスタム ビルドの Docker イメージの使用]({{ site.baseurl }}/2.0/custom-images/)」を参照してカスタム イメージを作成し、ジョブの要件に応じてコンテナに事前ロードされるツールを含めることをお勧めします。

More details on the Docker Executor are available in the [Configuring CircleCI]({{ site.baseurl }}/2.0/configuration-reference/) document.

### Using Multiple Docker Images

It is possible to specify multiple images for your job. Specify multiple images if, for example, you need to use a database for your tests or for some other required service. **In a multi-image configuration job, all steps are executed in the container created by the first image listed**. All containers run in a common network and every exposed port will be available on `localhost` from a [primary container]({{ site.baseurl }}/2.0/glossary/#primary-container).

```yaml
jobs:
  build:
    docker:
    # Primary container image where all steps run.
     - image: buildpack-deps:trusty
    # Secondary container image on common network. 
     - image: mongo:2.6.8-jessie
       command: [mongod, --smallfiles]

    working_directory: ~/

    steps:
      # command will execute in trusty container
      # and can access mongo on localhost

      - run: sleep 5 && nc -vz localhost 27017
```

Docker Images may be specified in three ways, by the image name and version tag on Docker Hub or by using the URL to an image in a registry:

#### Public Convenience Images on Docker Hub
{:.no_toc}
  * `name:tag` 
    * `circleci/node:7.10-jessie-browsers`
* `name@digest` 
    * `redis@sha256:34057dd7e135ca41...`

#### Public Images on Docker Hub
{:.no_toc}
  * `name:tag` 
    * `alpine:3.4`
* `name@digest` 
    * `redis@sha256:54057dd7e125ca41...`

#### Public Docker Registries
{:.no_toc}
  * `image_full_url:tag` 
    * `gcr.io/google-containers/busybox:1.24`
* `image_full_url@digest` 
    * `gcr.io/google-containers/busybox@sha256:4bdd623e848417d9612...`

Nearly all of the public images on Docker Hub and Docker Registry are supported by default when you specify the `docker:` key in your `config.yml` file. If you want to work with private images/registries, please refer to [Using Private Images]({{ site.baseurl }}/2.0/private-images).

### RAM disks

A RAM disk is available at `/mnt/ramdisk` that offers a [temporary file storage paradigm](https://en.wikipedia.org/wiki/Tmpfs), similar to using `/dev/shm`. Using the RAM disk can help speed up your build, provided that the `resource_class` you are using has enough memory to fit the entire contents of your project (all files checked out from git, dependencies, assets generated etc).

The simplest way to use this RAM disk is to configure the `working_directory` of a job to be `/mnt/ramdisk`:

```yaml
jobs:
  build:
    docker:
     - image: alpine

    working_directory: /mnt/ramdisk

    steps:

      - run: |
          echo '#!/bin/sh' > run.sh
          echo 'echo Hello world!' >> run.sh
          chmod +x run.sh
      - run: ./run.sh
```

### Docker Benefits and Limitations

Docker also has built-in image caching and enables you to build, run, and publish Docker images via \[Remote Docker\]\[building-docker-images\]. Consider the requirements of your application as well. If the following are true for your application, Docker may be the right choice:

* Your application is self-sufficient
* Your application requires additional services to be tested
* Your application is distributed as a Docker Image (requires using \[Remote Docker\]\[building-docker-images\])
* You want to use `docker-compose` (requires using \[Remote Docker\]\[building-docker-images\])

Choosing Docker limits your runs to what is possible from within a Docker container (including our \[Remote Docker\]\[building-docker-images\] feature). For instance, if you require low-level access to the network or need to mount external volumes consider using `machine`.

There are tradeoffs to using a `docker` image versus an Ubuntu-based `machine` image as the environment for the container, as follows:

| Capability                                                                            | `docker`           | `machine` |
| ------------------------------------------------------------------------------------- | ------------------ | --------- |
| 起動時間                                                                                  | 即時                 | 30 ～ 60 秒 |
| クリーン環境                                                                                | ○                  | ○         |
| カスタム イメージ                                                                             | Yes <sup>(1)</sup> | ×         |
| Docker イメージのビルド                                                                       | Yes <sup>(2)</sup> | ○         |
| ジョブ環境の完全な制御                                                                           | ×                  | ○         |
| 完全なルート アクセス                                                                           | ×                  | ○         |
| 複数データベースの実行                                                                           | Yes <sup>(3)</sup> | ○         |
| 同じソフトウェアの複数バージョンの実行                                                                   | ×                  | ○         |
| [Docker Layer Caching]({{ site.baseurl }}/2.0/docker-layer-caching/)                  | ○                  | ○         |
| 特権コンテナの実行                                                                             | ×                  | ○         |
| Docker Compose とボリュームの使用                                                              | ×                  | ○         |
| [構成可能なリソース (CPU/RAM)]({{ site.baseurl }}/2.0/configuration-reference/#resource_class) | ○                  | ○         |
{: class="table table-striped"}

<sup>(1)</sup> See \[Using Custom Docker Images\]\[custom-images\].

<sup>(2)</sup> Requires using \[Remote Docker\]\[building-docker-images\].

<sup>(3)</sup> While you can run multiple databases with Docker, all images (primary and secondary) share the underlying resource limits. Performance in this regard will be dictated by the compute capacities of your container plan.

For more information on `machine`, see the next section below.

### Available Docker Resource Classes

The [`resource_class`]({{ site.baseurl }}/2.0/configuration-reference/#resource_class) key allows you to configure CPU and RAM resources for each job. In Docker, the following resources classes are available:

| クラス                    | vCPU | RAM  |
| ---------------------- | ---- | ---- |
| small                  | 1    | 2GB  |
| medium (default)       | 2    | 4GB  |
| medium+                | 3    | 6GB  |
| large                  | 4    | 8GB  |
| xlarge                 | 8    | 16GB |
| 2xlarge<sup>(2)</sup>  | 16   | 32GB |
| 2xlarge+<sup>(2)</sup> | 20   | 40GB |
{: class="table table-striped"}

Where example usage looks like the following:

```yaml
jobs:
  build:
    docker:
      - image: buildpack-deps:trusty
    resource_class: xlarge
    steps:
    #  ...  other config
```

## Machine の使用

The `machine` option runs your jobs in a dedicated, ephemeral VM that has the following specifications:

{% include snippets/machine-resource-table.md %}

Using the `machine` executor gives your application full access to OS resources and provides you with full control over the job environment. This control can be useful in situations where you need full access to the network stack, for example to listen on a network interface, or to modify the system with `sysctl` commands. To find out about migrating a project from using the Docker executor to using `machine`, see the [Executor Migration from Docker to Machine]({{ site.baseurl }}/2.0/docker-to-machine) document.

Using the `machine` executor also means that you get full access to the Docker process. This allows you to run privileged Docker containers and build new Docker images.

**Note**: Using `machine` may require additional fees in a future pricing update.

To use the `machine` executor, set the [`machine` key]({{ site.baseurl }}/2.0/configuration-reference/#machine) to `true` in `.circleci/config.yml`:

{:.tab.machineblock.Cloud}
```yaml
version: 2
jobs:
  build:
    machine:
      image: ubuntu-1604:201903-01    # recommended linux image - includes Ubuntu 16.04, docker 18.09.3, docker-compose 1.23.1
```

{:.tab.machineblock.Server}
```yaml
version: 2
jobs:
  build:
    machine: true # uses default image
```

**Note:** The default image for the machine executor is `circleci/classic:latest`. If you don't specify an image, jobs will run on the default image - which is currently circleci/classic:201710-01 but may change in future.

All images have common language tools preinstalled. Refer to the [specification script for the VM](https://raw.githubusercontent.com/circleci/image-builder/picard-vm-image/provision.sh) for more information.

**Note:** The `image` key is not required on self-hosted installations of CircleCI Server (see example above) but if it is used, it should be set to: `image: default`.

The following example uses the default machine image and enables [Docker Layer Caching]({{ site.baseurl }}/2.0/docker-layer-caching) (DLC) which is useful when you are building Docker images during your job or Workflow.

```yaml
version: 2
jobs:
  build:
    machine:
      image: ubuntu-1604:201903-01
      docker_layer_caching: true    # default - false
```

## macOS の使用

*Available on CircleCI Cloud - not currently available on self-hosted installations*

Using the `macos` executor allows you to run your job in a macOS environment on a VM. You can also specify which version of Xcode should be used. See the [Supported Xcode Versions section of the Testing iOS]({{ site.baseurl }}/2.0/testing-ios/#supported-xcode-versions) document for the complete list of version numbers and information about technical specifications for the VMs running each particular version of Xcode.

```yaml
jobs:
  build:
    macos:
      xcode: 11.3.0

    steps:
      # Commands will execute in macOS container
      # with Xcode 11.3 installed

      - run: xcodebuild -version
```

## Windows Executor の使用

Using the `windows` executor allows you to run your job in a Windows environment. The following is an example configuration that will run a simple Windows job. The syntax for using the Windows executor in your config differs depending on whether you are using:

* CircleCI Cloud – config version 2.1.
* Self-hosted installation of CircleCI Server with config version 2.0 – this option is an instance of using the `machine` executor with a Windows image – *Introduced in CircleCI Server v2.18.3*.

{:.tab.windowsblock.Cloud}
```yaml
version: 2.1 # Use version 2.1 to enable Orb usage.

orbs:
  win: circleci/windows@2.2.0 # The Windows orb give you everything you need to start using the Windows executor.

jobs:
  build: # name of your job
    executor: win/default # executor type

    steps:
      # Commands are run in a Windows virtual machine environment

      - checkout
      - run: Write-Host 'Hello, Windows'
```

{:.tab.windowsblock.Server}
```yaml
version: 2

jobs:
  build: # name of your job
    machine:
      image: windows-default # Windows machine image
    resource_class: windows.medium
    steps:
      # Commands are run in a Windows virtual machine environment

        - checkout
        - run: Write-Host 'Hello, Windows'
```

Cloud users will notice the Windows Orb is used to set up the Windows executor to simplify the configuration. See [the Windows orb details page](https://circleci.com/orbs/registry/orb/circleci/windows) for more details.

CircleCI Server users should contact their system administrator for specific information about the image used for Windows jobs. The Windows image is configured by the system administrator, and in the CircleCI config is always available as the `windows-default` image name.

## Using GPUs

CircleCI Cloud has execution environments with Nvidia GPUs for specialized workloads. The hardware is Nvidia Tesla T4 Tensor Core GPU, and our GPU executors come in both Linux and Windows VMs.

{:.tab.gpublock.Linux}
```yaml
version: 2.1

jobs:
  build:
    machine:
      resource_class: gpu.nvidia.small 
      image: ubuntu-1604-cuda-10.1:201909-23
    steps:

      - run: nvidia-smi
```

{:.tab.gpublock.Windows}
```yaml
version: 2.1

orbs:
  win: circleci/windows@2.3.0

jobs:
  build:
    executor: win/gpu-nvidia
    steps:

      - run: '&"C:\Program Files\NVIDIA Corporation\NVSMI\nvidia-smi.exe"'
```

Customers using CircleCI server can configure their VM service to use GPU-enabled machine executors. See \[Running GPU Executors in Server\]\[server-gpu\].

## See Also

[Configuration Reference]({{ site.baseurl }}/2.0/configuration-reference/)