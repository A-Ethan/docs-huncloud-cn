A build stage, with a job called compile.
A test stage, with two jobs called test1 and test2.
A staging stage, with a job called deploy-to-stage.
A production stage, with a job called deploy-to-prod.


https://ethan.shen%40ucloud.cn:Sx19881104@git.ucloudadmin.com/kunpublic/cicd-demo.git

如果这个docs托管至gitlab，.gitlab-ci.yaml

```yaml
# 全局变量
variables:
  secrets.OSS_KEY_ID: aaa
  secrets.OSS_KEY_SECRET: bbb

# stages 顺序运行, 同一个 stage 的所有 job 并行
stages:
  - Check
  - Build
  - Deploy # Dev开发 Test测试 Pre灰度
  - ProductionDeploy # 生产部署

# 使用lint 来进行代码静态检查，待补充
lint:
  stage: Check
  tags:
    - dev
    - master
  image: hub.ucloudadmin.com/public/golangci-lint:latest
  script:
    - ./main

# 单元测试
unit-test:
  stage: Check
  tags:
    - dev
    - master
  # 如果一个 job 的运行，依赖其他的服务，可以作为 service 添加进去
  # 这里我们依赖 redis 来运行单元测试
  # redis 容器将会运行在同一个 k8s pod 中，这个 job 中，pod 将包含 3 个容器：
  #   1. golang 镜像的容器来运行单元测试
  #   2. helper 镜像容器来把代码从 gitlab 上拉下来
  #   3. redis 镜像容器
  # 这 3 个容器共享一个 network namespace（拥有用一个 IP 地址）
  services:
    - redis:4.0.9
  image: golang:1.14.2
  script:
    - "cmd"
  # coverage 指令定义了一个正则表达式，gitlab 会使用该正则匹配这个 job 的输出，作为“覆盖率”的数据显示在 web 界面上
  coverage: '/coverage: \d+.\d+% of statements/'


```

```yaml

# stages 顺序运行, 同一个 stage 的所有 job 并行
stages:
  - Check
  - BuildImage
  - PreDeploy
  - ProductionDeploy

# 使用 golaing-ci lint 来进行代码静态检查，具体参考 https://git.ucloudadmin.com/kunpublic/golangci-lint
lint:
  image: hub.ucloudadmin.com/public/golangci-lint:latest
  # 通过 uaek 的标签来使用 KUN 真共享 Runner
  tags:
    - uaek
  stage: Check
  script:
    # Use default .golangci.yml file from the image if one is not present in the project root.
    - '[ -e .golangci.yml ] || cp /golangci/.golangci.yml .'
    - golangci-lint run

unit-test:
  stage: Check
  # 通过 uaek 的标签来使用 KUN 真共享 Runner
  tags:
    - uaek
  image: golang:1.14.2
  # 如果一个 job 的运行，依赖其他的服务，可以作为 service 添加进去
  # 这里我们依赖 redis 来运行单元测试
  # redis 容器将会运行在同一个 k8s pod 中，这个 job 中，pod 将包含 3 个容器：
  #   1. golang 镜像的容器来运行单元测试
  #   2. helper 镜像容器来把代码从 gitlab 上拉下来
  #   3. redis 镜像容器
  # 这 3 个容器共享一个 network namespace（拥有用一个 IP 地址）
  services:
    - redis:4.0.9
  script:
    - mkdir -p /go/src/git.ucloudadmin.com/labu-public
    # $CI_PROJECT_DIR 就是代码的目录
    - cp -r $CI_PROJECT_DIR /go/src/git.ucloudadmin.com/labu-public/uaek-cicd-demo
    - go test -mod vendor -cover -v git.ucloudadmin.com/labu-public/uaek-cicd-demo/...
  # coverage 指令定义了一个正则表达式，gitlab 会使用该正则匹配这个 job 的输出，作为“覆盖率”的数据显示在 web 界面上
  coverage: '/coverage: \d+.\d+% of statements/'


# 下面构建一个 docker 镜像
# Kaniko 允许我们在不需要任何特权的情况下，在容器中构建 docker 镜像
#
# 示例:
# use docker:
#     cd /path/to/project && docker build -t uhub.service.ucloud.cn/myimage:0.0.1 -f deploy/Dockerfile
# use Kaniko:
#     /kaniko/executor -c /path/to/project -f deploy/Dockerfile -d uhub.service.ucloud.cn/myimage:0.0.1
#
# Usage:
#   executor [flags]
#   executor [command]
# 
# Available Commands:
#   help        Help about any command
#   version     Print the version number of kaniko
# 
# Flags:
#       --build-arg multi-arg type                  This flag allows you to pass in ARG values at build time. Set it repeatedly for multiple values.
#       --cache                                     Use cache when building image
#       --cache-dir string                          Specify a local directory to use as a cache. (default "/cache")
#       --cache-repo string                         Specify a repository to use as a cache, otherwise one will be inferred from the destination provided
#       --cache-ttl duration                        Cache timeout in hours. Defaults to two weeks. (default 336h0m0s)
#       --cleanup                                   Clean the filesystem at the end
#   -c, --context string                            Path to the dockerfile build context. (default "/workspace/")
#       --context-sub-path string                   Sub path within the given context.
#   -d, --destination multi-arg type                Registry the final image should be pushed to. Set it repeatedly for multiple destinations.
#       --digest-file string                        Specify a file to save the digest of the built image to.
#   -f, --dockerfile string                         Path to the dockerfile to be built. (default "Dockerfile")
#       --force                                     Force building outside of a container
#   -h, --help                                      help for executor
#       --image-name-with-digest-file string        Specify a file to save the image name w/ digest of the built image to.
#       --insecure                                  Push to insecure registry using plain HTTP
#       --insecure-pull                             Pull from insecure registry using plain HTTP
#       --insecure-registry multi-arg type          Insecure registry using plain HTTP to push and pull. Set it repeatedly for multiple registries.
#       --label multi-arg type                      Set metadata for an image. Set it repeatedly for multiple labels.
#       --log-format string                         Log format (text, color, json) (default "color")
#       --log-timestamp                             Timestamp in log output
#       --no-push                                   Do not push the image to the registry
#       --oci-layout-path string                    Path to save the OCI image layout of the built image.
#       --registry-certificate key-value-arg type   Use the provided certificate for TLS communication with the given registry. Expected format is 'my.registry.url=/path/to/the/server/certificate'.
#       --registry-mirror string                    Registry mirror to use has pull-through cache instead of docker.io.
#       --reproducible                              Strip timestamps out of the image to make it reproducible
#       --single-snapshot                           Take a single snapshot at the end of the build.
#       --skip-tls-verify                           Push to insecure registry ignoring TLS verify
#       --skip-tls-verify-pull                      Pull from insecure registry ignoring TLS verify
#       --skip-tls-verify-registry multi-arg type   Insecure registry ignoring TLS verify to push and pull. Set it repeatedly for multiple registries.
#       --skip-unused-stages                        Build only used stages if defined to true. Otherwise it builds by default all stages, even the unnecessaries ones until it reaches the target stage / end of Dockerfile
#       --snapshotMode string                       Change the file attributes inspected during snapshotting (default "full")
#       --tarPath string                            Path to save the image in as a tarball instead of pushing
#       --target string                             Set the target build stage to build
#       --use-new-run                               Experimental run command to detect file system changes. This new run command does no rely on snapshotting to detect changes.
#   -v, --verbosity string                          Log level (trace, debug, info, warn, error, fatal, panic) (default "info")
#       --whitelist-var-run                         Ignore /var/run directory when taking image snapshot. Set it to false to preserve /var/run/ in destination image. (Default true). (default true)
# 
# Use "executor [command] --help" for more information about a command.
docker-image:
  # 通过 uaek 的标签来使用 KUN 真共享 Runner
  tags:
    - uaek
  stage: BuildImage
  # 请使用 UAEK Kaniko 镜像，该 Image 在官方镜像基础上，添加了 busybox，这样才能和 gitlab 联动起来
  image: hub.ucloudadmin.com/public/uaek-kaniko-executor:latest
  script:
    # 首先我们需要为我们的镜像生成一个 tag，规则是：如果有 git tag，就使用 git tag，如果没有的话，就使用 git commit sha
    - IMAGE_TAG=$CI_COMMIT_SHA && if [[ -n "$CI_COMMIT_TAG" ]]; then IMAGE_TAG=$CI_COMMIT_TAG ; fi
    # - /kaniko/executor -c $CI_PROJECT_DIR -f build/Dockerfile --cache=true -d $IMAGE_NAME:$IMAGE_TAG --cache-repo ${IMAGE_NAME}-cache
    - /kaniko/executor -c $CI_PROJECT_DIR -f build/Dockerfile -d $IMAGE_NAME:$IMAGE_TAG
  # 使用 only 来限制这个 job 什么情况下会运行，下面的设置标识只有在 dev 分支发生变化，或者新的 tag 被创建时才会运行。
  only:
    - tags
    - dev
    - master


# 预发布，将部署在 uae-a3 集群的 prj-labu-test 项目下，资源集为 uaek-cicd-demo-test
pre-deploy:
  tags:
    - uaek
  stage: PreDeploy
  # 我们使用 uaek-ciclient 工具镜像，在 UAEK 的部署系统中创建一个版本，该版本属于资源集 uaek-cicd-demo-test
  image: hub.ucloudadmin.com/uaek/uaek-ciclient:latest
  variables:
    # uaek-ciclient 工具镜像将使用下面 7 个环境变量
    CD_PROJECT: prj-labu-test           # UAEK 项目名
    CD_RESOURCESET: uaek-cicd-demo-test # 资源集名称
    CD_FILE: deploy/output/pre.yaml     # YAML 文件，将会在下面使用 kustomize 工具生成
    CD_USERNAME: $KUN_USER              # 用户名（邮箱前缀）
    CD_PASSWORD: $KUN_PASSWORD          # UAEK 授权码
    CD_CREATE_JOB: "true"               # true 或 false，表示是否自动创建任务 ，如果设置为 true，会创建一个该版本的部署任务，即资源将会部署在集群中
    CD_CLUSTER: uae-a3
  script:
    - IMAGE_TAG=$CI_COMMIT_SHA && if [[ -n "$CI_COMMIT_TAG" ]]; then IMAGE_TAG=$CI_COMMIT_TAG ; fi
    - cd $CI_PROJECT_DIR/deploy/overlays/pre && kustomize edit set image $IMAGE_NAME=$IMAGE_NAME:$IMAGE_TAG
    - cd $CI_PROJECT_DIR && mkdir deploy/output && kustomize build deploy/overlays/pre > ${CD_FILE}
    - cd $CI_PROJECT_DIR && /root/ciclient -version=$IMAGE_TAG
  # argifacts will be uploaded to gitlab, you can download them from browser
  artifacts:
    paths:
      - deploy/output
  # this job will run only on dev branch OR new tag is created
  only:
    - tags
    - dev


# 正式发布，将部署在 uae-a3 集群的 prj-labu 项目下，资源集为 uaek-cicd-demo
production-deploy:
  tags:
    - uaek
  stage: ProductionDeploy
  image: hub.ucloudadmin.com/uaek/uaek-ciclient:latest
  variables:
    CD_PROJECT: prj-labu
    CD_RESOURCESET: uaek-cicd-demo
    CD_FILE: deploy/output/production.yaml
    CD_USERNAME: $KUN_USER              # 用户名（邮箱前缀）
    CD_PASSWORD: $KUN_PASSWORD          # UAEK 授权码
    CD_CREATE_JOB: "false"  # 设置为 false，部署任务不会被自动创建，用户需要在 KUN 控制台上手工创建
  script:
    - IMAGE_TAG=$CI_COMMIT_SHA && if [[ -n "$CI_COMMIT_TAG" ]]; then IMAGE_TAG=$CI_COMMIT_TAG ; fi
    - cd $CI_PROJECT_DIR/deploy/overlays/production && kustomize edit set image $IMAGE_NAME=$IMAGE_NAME:$IMAGE_TAG
    - cd $CI_PROJECT_DIR && mkdir deploy/output && kustomize build deploy/overlays/production > ${CD_FILE}
    - cd $CI_PROJECT_DIR && /root/ciclient -version=$IMAGE_TAG
  # argifacts will be uploaded to gitlab, you can download them from browser
  artifacts:
    paths:
      - deploy/output
  # 我们不希望正式部署全自动运行，所以这里指定了 when: manual，只有当人工在页面上点击了继续之后，这个 job 才会运行
  when: manual
  # allow_failure 规定了这个 job 是否允许失败，也就是说当这个 job 失败时，下面的 job 是否运行，默认值为 false
  # 但是当指定了 when: manual 后，allow_failure 会变成 true，所以我们显式地加上 allow_failure: false
  allow_failure: false
  # run this job only when new tag is created
  only:
    - tags

```


```yaml

variables:
  MAVEN_CLI_OPTS: "--batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths:
    - .m2/repository/
    - target/

buildToIntegraion:
  stage: build
  only:
    refs:
      - develop
  script:
    - mvn $MAVEN_CLI_OPTS package
    - sudo docker build -t uhub.ucloud.cn/gary/hello-image:SNAP-$CI_PIPELINE_ID .


buildToDeploy:
  stage: build
  only:
    - tags
  script:
    - mvn $MAVEN_CLI_OPTS package
    - sudo docker build -t uhub.ucloud.cn/gary/hello-image:$CI_COMMIT_TAG .
    - sudo docker push uhub.ucloud.cn/gary/hello-image:$CI_COMMIT_TAG
test:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS test


deployToIntegration:
  only:
    refs:
      - develop
  stage: deploy
  script:
    - sudo docker push uhub.ucloud.cn/gary/hello-image:SNAP-$CI_PIPELINE_ID
    - cat  yaml/hello_world_template.yaml |sed "s/<IMAGE_TAG>/SNAP-$CI_PIPELINE_ID/"  > /tmp/hello_world_SNAP-$CI_PIPELINE_ID.yaml
    - scp /tmp/hello_world_SNAP-$CI_PIPELINE_ID.yaml root@k8s:/tmp/hello_world_SNAP-$CI_PIPELINE_ID.yaml
    - ssh root@k8s kubectl apply -f /tmp/hello_world_SNAP-$CI_PIPELINE_ID.yaml

deployToPre:
  only:
    - tags
  stage: deploy
  script:
    - cat  yaml/hello_world_template.yaml |sed "s/<IMAGE_TAG>/$CI_COMMIT_TAG/"  > /tmp/hello_world_$CI_COMMIT_TAG.yaml
    - scp /tmp/hello_world_$CI_COMMIT_TAG.yaml root@k8s:/tmp/hello_world_$CI_COMMIT_TAG.yaml
    - ssh root@k8s kubectl apply -f /tmp/hello_world_$CI_COMMIT_TAG.yaml -n preproduction

deployToProduct:
  only:
    - tags
  stage: deploy
  when: manual
  script:
    - cat  yaml/hello_world_template.yaml |sed "s/<IMAGE_TAG>/$CI_COMMIT_TAG/"  > /tmp/hello_world_$CI_COMMIT_TAG.yaml
    - scp /tmp/hello_world_$CI_COMMIT_TAG.yaml root@k8s:/tmp/hello_world_$CI_COMMIT_TAG.yaml
    - ssh root@k8s kubectl apply -f /tmp/hello_world_$CI_COMMIT_TAG.yaml -n production

```