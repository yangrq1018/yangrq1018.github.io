---
title: Gitlab CI 使用笔记
toc:
  number: false
  enable: true
tags:
- Git
- Gitlab
- Continuous Integration
- Docker
categories: 技术
permalink: article/gitlab-ci-memo
description: 自动化就是一切！
---
##  Gitlab CI 使用笔记

### Gitlab CI是如何帮助我们的项目的
Python开发的项目，除了享受Python简介的语法和丰富的第三方库之外，会遇到几个痛点
- 动态类型带来的问题
  - 维护难度大
  - 没有编译环节，错误只能在运行时发现
- 第三方库，甚至包括Python本身的迭代速度快，不兼容更新是家常便饭，由此导致库的依赖较难管理

解决第一个痛点，常见的作法是
- 使用3.7首次引入的`typing`模块，手动给代码加上类型，这其实跟TypeScript很像，而TypeScript的成功
证明`typing`未来一定是Python不可或缺的一部分，类型标记带来的便利肯定会让Python在大型项目中的实用性
得到很大的提升
- 单元测试覆盖，让bug在测试这个环节尽可能的显现出来

解决第二个痛点，有常见的这些方案
- 众所周知，Python库的相互依赖已经到了一个可以说是一团糟的地步，`pip`在早先版本中，几乎没有处理库的
依赖关系的功能，这也是不建议你在复杂的环境中使用`pip install`的原因，如果你的项目里引用了20个库，其中
包括了Panda或者Numpy这样庞大的库，那你的下一个`pip install XXX`是有很大可能break掉你的代码的。`pip`在
20.X版本后加入了新的Dependency resolve模块，可能会让这样的烦恼少一些。但我们现在就可以使用`Pipenv`代替
`pip`，管理我们的项目依赖，一个`Pipfile`的作用，参见JavaScript里的`packages.json`。
- Conda也是一个较成熟的Python环境管理工具，Anaconda Python分发版已经是很多人的Python首选了，使用`conda install`
会让Anaconda的官方repository来帮你解析依赖关系

### 使用持续集成CI完成编译-测试-发布流程
我司有一个假设在内网里的私有代码仓库，使用的是Gitlab，用Nginx做反向代理后，可以在内网里通过
https://gitlab.xuanlingasset.com访问。

Gitlab中的每个项目都有一个CI/CD区域，这个区域的功能是定义一些你想要经常运行的编译/测试/发布...的（无聊）工作，让
Gitlab来帮你执行，比如你经常有这样一个需求:

经过一天的代码修改，你提交了一个commit，把commit push到Gitlab服务器上，这时你想要运行所有的单元测试，确保你新更改的
代码没有破坏原有的功能，或者引入了一些bug。如果单元测试通过的话，把项目编译打包后发布。

这时一个`.gitlab-ci.yml`文件就可以帮到你了。Gitlab CI，简单来说就是在Gitlab服务器之外，你可以建立一个`gitlab-runner`
服务器，这个服务器负责代理一个容器管理工具，比如Docker/Virtual Box等，我们选用的是Docker。commit和tag这样的时间可以
触发了CI工作流，假设你push了一个commit到Gitlab服务器上，runner就会探测到需要运行相关的工作流，它建立一个新的Docker容器，
从Gitlab上拉取新的代码，然后运行预先定义的scripts，如果没有错误产生，工作流就成功完成了。

### 定义Gitlab CI流程
假设我们的项目结构是这样的:
```
- foo
 |- src
 |- tests
 |- .gitlab-ci.yml
 |- Pipfile
 |- setup.py
```

`.gitlab-ci.yml`的一个例子
```yaml
image: local/python:3.8.7

# Change pip's cache directory to be inside the project directory since we can
# only cache local items.
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  # the local/python:3.8.7 has the correct CA install in system
  REQUESTS_CA_BUNDLE: "/etc/ssl/certs/ca-certificates.crt"

cache:
  paths:
    - .cache/pip
    - venv/

stages:
  - build
  - upload
  - release

build_job:
  stage: build
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'
  script:
    # activate virtualenv
    - pip install virtualenv
    - virtualenv venv
    - source venv/bin/activate

    # force fetch, update tags
    - git config fetch.prune true
    - git config fetch.pruneTags true
    - git fetch --tags -f
    - git tag

    # build and install
    - python setup.py bdist_wheel
    - pip install dist/*
  artifacts:
    paths:
      - dist/*.whl

upload_job:
  stage: upload
  image: local/python:3.8.7
  rules:
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'
  script:
    - pip install requests
    - pip install packaging # extract base version
    - pip install simplejson
    - python -m py_ci upload

release_job:
  stage: release
  rules:
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'
  script:
    - pip install requests
    - pip install packaging
    - pip install simplejson
    - python -m py_ci release
```


下面是详细解释
```yaml
stages:
  - build
  - upload
  - release
```
定义了工作流中的顺序，三个阶段build -> upload -> release

第一个job：初始化环境，安装需要的包
```yaml
build_job:
  stage: build
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'  # 当commit被推送到master branch上时这个工作会触发
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'  # 当commit带有一个'release/'开头的tag时触发
  script:
    # 创建一个虚拟环境，好处是我们可以缓存pip安装的包，下次执行这个工作的时候不需要从头安装这些库
    - pip install virtualenv
    - virtualenv venv
    - source venv/bin/activate

    # 从gitlab上拉取最新的tag，注意没有这段的话runner只会拉取最新的代码，不会拉取tag
    # 我们需要tag的原因是`versioneer`在git历史中查找tag来生产当前的__version__
    - git config fetch.prune true
    - git config fetch.pruneTags true
    - git fetch --tags -f
    - git tag

    # 编译打包项目
    - python setup.py bdist_wheel
    - pip install dist/*
  # artifacts中指定的文件会被工作流中之后的工作继承
  artifacts:
    paths:
      - dist/*.whl
```

如果你有一些单元测试，在这一步可以加入一个test阶段
```yaml
stages:
  - build
  - test
  - upload
  - release

test_job:
  stage: test
  script:
   - python setup.py test
   # 如果你用的时pytest的话
   - pytest test/
```

test全部成功的话，test阶段会成功返回，开始下一个upload工作，把打包好的项目wheel文件上传到该Gitlab仓库的Package registry里
```yaml
upload_job:
  stage: upload
  image: local/python:3.8.7
  rules:
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'
  script:
    - pip install requests
    - pip install packaging # extract base version
    - pip install simplejson
    - python -m py_ci upload
```

我们可以在CI工作中调用Gitlab的API，这也是最简单使用的方法，你可以用任何语言做HTTP请求，我们自己写了一个Python帮助脚本来执行
upload，也就是这一步`python -m py_ci upload`。

`py_ci`的内容很简答，`py_ci/upload.py`的内容
```py3
import os

import requests

from .helper import PACKAGE_URL, WHEEL_FILE_NAME

def upload():
    print('Post target: {}'.format(PACKAGE_URL))
    dist_path = 'dist/{}'.format(WHEEL_FILE_NAME)
    files = {'upload_file': open(dist_path, 'rb')}
    print('Upload file: {}'.format(dist_path))

    headers = {
        "JOB-TOKEN": os.environ['CI_JOB_TOKEN']  # authenciate with Gitlab API
    }
    res = requests.put(PACKAGE_URL,
                       headers=headers,
                       files=files
                       )
    print(res.content.decode())
    if res.status_code == 201:
        print('Package upload to package registry (generic)!')
    else:
        raise RuntimeError(res.status_code)
```
简单来说就是把`dist/`下的whl文件put到相应的url上，就完成了package的上传。
> Gitalb的API访问需要权限，权限验证有很多种方法，这里的`JOB-TOKEN`是CI容器中自动定义的环境变量，可以用来验证权限；如果你需要在
> 其他地方调用API，可以在项目的页面生成一个Personal token

最后一步，建立一个Release
```yaml
release_job:
  stage: release
  rules:
    - if: '$CI_COMMIT_TAG =~ /release\/(.*)/'
  script:
    - pip install requests
    - pip install packaging
    - pip install simplejson
    - python -m py_ci release
```

`py_ci/release.py`的内容
```py3
import os

import requests
import simplejson

from .helper import PACKAGE_URL, TAG_NAME, WHEEL_FILE_NAME, CI_API_V4_URL, CI_PROJECT_ID, FULL_VERSION


def release():
    post_target = CI_API_V4_URL + f"/projects/{CI_PROJECT_ID}/releases"
    headers = {
        'JOB-TOKEN': os.environ['CI_JOB_TOKEN'],
        'Content-Type': 'application/json'
    }
    payload = simplejson.dumps(release_data())
    res = requests.post(post_target,
                        data=payload,
                        headers=headers
                        )
    print('Response code: {}'.format(res.status_code))
    print(res.content.decode())
    if res.status_code == 201:
        print('Release succesful! Receipt:')
        print(res.content)
    else:
        raise ValueError(res.status_code)


def release_data():
    data = {
        "name": f"Release {FULL_VERSION}",
        "tag_name": TAG_NAME,
        "description": f"Version {FULL_VERSION}",
        # "milestones": "" # if specified must exist
        "assets": {
            "links": [
                {
                    "name": WHEEL_FILE_NAME,
                    "url": PACKAGE_URL
                }
            ]
        }
    }
    return data
```
我们按照Gitlab API的要求，在`release_data`函数中提供了必要的release信息，最重要的是`assets"字段：
`assets`字段是选填的，如果你不提供的话，产生的release只会包含一份repo内源码的打包，那我们怎么让release中也包含上一步
upload里上传的whl文件呢？只需要提供一个URL连接到package registry里的对应whl文件的地址就可以


### 私有CA证书
在部署Runner的时候，遇到了一个比较棘手的问题：我们的Gitlab实例只能在内网里可见，因此无法取得一个公开的SSL证书，我们只得
使用一张自签发证书来加密Gitlab服务器，但是CI里的Docker容器环境里没有这张私有证书，会导致无法和Gitlab服务器通讯。

Gitlab官方给出了[解决方法](https://docs.gitlab.com/runner/configuration/tls-self-signed.html)，简单来说就是指定一个
`tls-ca-file`配置给Docker容器，然后Docker容器会从该文件读取CA证书，遗憾的是这个方法经测试不能奏效。

我们在`python:3.8.7-alpine`官方镜像上，叠加了一层，把自签名的CA证书打包到镜像中，因此你可以看到`.gitlab-ci.yml`中
```
image: local/python:3.8.7
```
用的是这张本地镜像，`local/python:3.8.7`和普通的python镜像在功能上没有任何区别
> 如果你的Gitlab服务器用的是经过认证的公有证书，那你完全可以使用其他任何的Docker镜像，`local/python:3.8.7`只是为了解决
> SSL认证的无奈之举