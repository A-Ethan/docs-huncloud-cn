# Github Action

## 本文档自动流水线

```yaml
name: publish_build_gitbook_image
on: 
  push: #推送
    branches: #分支
    - main
jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
    # 切代码到 runner
    - name: checkout action
      uses: actions/checkout@v2
    # 使用 node:10
    - name: use Node.js 10.x
      uses: actions/setup-node@v1
      with:
        node-version: 10.x
    # npm install gitbook
    - name: npm install gitbook
      run: npm install -g gitbook-cli
    # git clone
    - uses: srt32/git-actions@v0.0.3
      with:
        args: git clone https://github.com/A-Ethan/docs-huncloud-cn.git  #git clone
    # cd 
    - name: cd ./docs-huncloud-cn
      run: cd ./docs-huncloud-cn
    # gitbook install & build
    - name: gitbook install
      run: gitbook install
    - name: gitbook build
      run: gitbook build ./ ./docs
    # 设置阿里云OSS的 id/secret，存储到 github 的 secrets 中
    - name: setup aliyun oss
      uses: A-Ethan/setup-ossutil@master
      with:
        endpoint: oss-cn-shanghai.aliyuncs.com
        access-key-id: ${{ secrets.OSS_KEY_ID }}
        access-key-secret: ${{ secrets.OSS_KEY_SECRET }}
    - name: cp files to aliyun
      run: ossutil cp -rf ./docs/ oss://docs-huncloud-cn/
```
