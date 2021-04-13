### 基本操作
- git conifg --list 查看所有配置信息
- git config --global user.name "" 配置全局用户名
- git config --global user.email "" 配置全局邮箱
- git init 初始化本地仓库（到创建的仓库文件夹下面执行）
- git add [fileName] 把文件放入暂存区
- git status 查看工作区文件状态
- git commit [fileName] -m [描述信息] 把暂存区的文件提交到节点

### 时光机穿梭
- git log 可以查看提交的记录
- git log --pretty=oneline 可以更简便的看出提交记录
- git reset --hard [HEAD^,commitid,HEAD~1] 版本回退
- git reflog 记录每一次的命令[可以找到以前提交的id,从而恢复版本]
- git diff HEAD -- [fileName] 可以查看工作区和版本库里面最新版本的区别：
- git checkout -- [fileName] 可以丢弃工作区的修改
- rm [fileName] 删除工作区的文件
- git rm [fileName] (并且commit) 删除版本库的文件

### 远程仓库

```
将本地仓库同步到远程仓库（使用码云）
1、在本地生成SSH密钥
    ssh-keygen -t rsa -C 'xxxx@xx.com'
    可以查看cat ~/.ssh/id_rsa.pub
2、把码云平台配置好密钥
3、本地仓库执行
    git remote add mayun git@gitee.com:sulin520/Demo1.git #创建一个远程仓库关联 名字为mayun
    git remote -v #可以查看目前远程关联的连接
    git remote rm [name] 可以删除一个远程连接
    git push -u mayun master

clone远程仓库
1、git clone 远程地址 保存本地位置


远程仓库同步 
git push mayun master
git pull mayun master
```
