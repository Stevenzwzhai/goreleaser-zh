---
title: 持续集成
menu: true
weight: 140
---

GoReleaser的第一次commit的构建思想，是作为CI集成的一部分运行.

让我们看看如何让它在流行的CI软件上运行.

## Travis CI

您可能希望将项目设置为新tag，就自动[Travis](https://travis-ci.org)部署, 例如:

```yaml
# .travis.yml
language: go

addons:
  apt:
    packages:
    # needed for the nfpm pipe:
    - rpm
    # needed for the snap pipe:
    - snapcraft

env:
# needed for the snap pipe:
- PATH=/snap/bin:$PATH

install:
# needed for the snap pipe:
- sudo snap install snapcraft --classic

# needed for the docker pipe
services:
- docker

after_success:
# docker login is required if you want to push docker images.
# DOCKER_PASSWORD should be a secret in your .travis.yml configuration.
- test -n "$TRAVIS_TAG" && docker login -u=myuser -p="$DOCKER_PASSWORD"
# snapcraft login is required if you want to push snapcraft packages to the
# store.
# You'll need to run `snapcraft export-login snap.login` and
# `travis encrypt-file snap.login --add` to add the key to the travis
# environment.
- test -n "$TRAVIS_TAG" && snapcraft login --with snap.login

# calls goreleaser
deploy:
- provider: script
  skip_cleanup: true
  script: curl -sL https://git.io/goreleaser | bash
  on:
    tags: true
    condition: $TRAVIS_OS_NAME = linux
```

注意最后一行(`condition: $TRAVIS_OS_NAME = linux`): 如果您运行具有多个Go版本和/或多个操作系统的构建矩阵，这一点很重要。如果是这种情况,您将需要确保GoReleaser只运行一次。

## CircleCI

这是如何与[CircleCI 2.0](https://circleci.com)协作:

```yml
# .circleci/config.yml
version: 2
jobs:
  release:
    docker:
      - image: circleci/golang:1.10
    steps:
      - checkout
      - run: curl -sL https://git.io/goreleaser | bash
workflows:
  version: 2
  release:
    jobs:
      - release:
          filters:
            branches:
              ignore: /.*/
            tags:
              only: /v[0-9]+(\.[0-9]+)*(-.*)*/
```

## Drone

默认情况下,Drone不会获取标签。`plugins/git`与默认值一起使用,在大多数情况下，我们需要覆盖`clone`步骤，启用标签，以使`goreleaser`工作正常.

在这个例子中，我们每次推送新标签时，都会创建一个新版本.请注意,您需在**repo settings**中启用`tags`和添加`github_token`密钥.

```yml
pipeline:
  clone:
    image: plugins/git
    tags: true

  test:
    image: golang:1.10
    commands:
      - go test ./... -race

  release:
    image: golang:1.10
    secrets: [github_token]
    commands:
      curl -sL https://git.io/goreleaser | bash
    when:
      event: tag
```

## Google CloudBuild

CloudBuild使用的克隆与你github repo不同: 似乎你的更改被拉取到了像下面`source.developers.google.com/p/YourProjectId/r/github-YourGithubUser-YourGithubRepo`类似的repo，和这就是你正在建设的东西.

这个repo具有糟糕的名称，所以为了防止Goreleaser发布到错误的github repo。请输入你的信息，到`.goreleaser.yml`文件的release部分:

```yml
release:
  github:
    owner: YourGithubUser
    name: YourGithubRepo
```

创建两个构建触发器:

-   常规CI的"推送到任何分支"触发器(不调用goreleaser)
-   一个"push to tag"触发器，它调用goreleaser

推送到任何分支触发器可以使用`Dockerfile`或`cloudbuild.yaml`,您喜欢就好.

你应该有一个专用的cloudbuild.release.yaml,它只能被"push to tag"触发器使用.

在这个例子中,我们每次推送新标签时，都会创建一个新版本.看看[使用加密资源](https://cloud.google.com/cloud-build/docs/securing-builds/use-encrypted-secrets-credentials)文档，其中有如何加密和base64编码你的github令牌.

构建使用的克隆[时没有标签](https://issuetracker.google.com/u/1/issues/113668706)，这就是为什么，我们必须明确运行`git tag $TAG_NAME`的原因( 请注意,只有当你的构建由"push to tag"触发时,才设置$TAG_NAME。) 这将允许goreleaser使用该版本创建一个版本，但它并**不会**构建一个适当只包含自先前标记以来提交的消息的**changelog**,.

```yml
steps:
~ # Setup the workspace so we have a viable place to point GOPATH at.
~ - name: gcr.io/cloud-builders/go
~   env: ['PROJECT_ROOT=github.com/YourGithubUser/YourGithubRepo']
~_  args: ['env']

~ # Create github release.
~ - name: goreleaser/goreleaser
~   entrypoint: /bin/sh
~   dir: gopath/src/github.com
~   env: ['GOPATH=/workspace/gopath']
~   args: ['-c', 'cd YourGithubUser/YourGithubRepo && git tag $TAG_NAME && /goreleaser' ]
~_  secretEnv: ['GITHUB_TOKEN']

  secrets:
~ - kmsKeyName: projects/YourProjectId/locations/global/keyRings/YourKeyRing/cryptoKeys/YourKey
~   secretEnv:
~     GITHUB_TOKEN: |
~       ICAgICAgICBDaVFBZUhVdUVoRUtBdmZJSGxVWnJDZ0hOU2NtMG1ES0k4WjF3L04zT3pEazhRbDZr
~       QVVTVVFEM3dVYXU3cVJjK0g3T25UVW82YjJaCiAgICAgICAgREtBMWVNS0hOZzcyOUtmSGoyWk1x
~_      ICAgICAgIEgwYndIaGUxR1E9PQo=
```

## Semaphore

在[Sempahore 2.0](https://semaphoreci.com)的每个项目，都在`.semaphore/semaphore.yml`指定默认管道的开头.

```yml
# .semaphore/semaphore.yml.
version: v1.0
name: Build
agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu1804

blocks:
  - name: "Test"
    task:
      prologue:
        commands:
          # set go version
          - sem-version go 1.11
          - "export GOPATH=~/go"
          - "export PATH=/home/semaphore/go/bin:$PATH"
          - checkout

      jobs:
        - name: "Lint"
          commands:
            - go get ./...
            - go test ./...

# On Semaphore 2.0 deployment and delivery is managed with promotions,
# which may be automatic or manual and optionally depend on conditions.
promotions:
    - name: Release
       pipeline_file: goreleaser.yml
       auto_promote_on:
         - result: passed
           branch:
             - "^refs/tags/v*"
```

`.semaphore/goreleaser.yml`中的管道文件:

```yml
version: "v1.0"
name: GoReleaser
agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu1804
blocks:
  - name: "Release"
    task:
      secrets:
        - name: goreleaser
      prologue:
        commands:
          - sem-version go 1.11
          - "export GOPATH=~/go"
          - "export PATH=/home/semaphore/go/bin:$PATH"
          - checkout
      jobs:
      - name: goreleaser
        commands:
          - curl -sL https://git.io/goreleaser | bash
```

下面的YAML文件,`createSecret.yml`创建一个名为goreleaser的密钥项，其包含一个名为**GITHUB_TOKEN**的环境变量:

```yml
apiVersion: v1alpha
kind: Secret
metadata:
  name: goreleaser
data:
  env_vars:
    - name: GITHUB_TOKEN
      value: "4afk4388304hfhei34950dg43245"
```

查看[管理密钥](https://docs.semaphoreci.com/article/15-secrets)的更详细文档.
