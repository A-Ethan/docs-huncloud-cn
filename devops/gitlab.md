# Gitlab

## git 命令

自己常用的也就这些了

```bash
# git 全局设置
git config --global user.name "A-Ethan"
git config --global user.email "200839608@qq.com"

# git clone
git clone http://github.com/a-ethan/docs-huncloud-cn.git
# git clone password
git clone http://200839608%40qq.com:password@github.com/a-ethan/docs-huncloud-cn.git

# add 暂存区
git add .

# commit 仓库区
git commit -m "update"

# push 远程仓库
git push -u origin master

# push 所有分支
git push -u orgin --all

# 列分支
git branch -a

# 新建分支，更新工作区
git checkout -b v1

# 切分支，更新工作区
git checkout master

# 合并指定分支到当前分支
git merge v1
```