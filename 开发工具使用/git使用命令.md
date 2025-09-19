### git常用命令

#### 1、创建库

```javascript
git init                          // 项目初始化
1
```

#### 2、添加和提交到仓库

```javascript
git add README.txt                // 添加 
git status                        // 提交前查看状态
git commit -m "name"       		  // 提交
git status                        // 提交后查看仓库状态
git diff readme.txt 			  // 查看文件更改前后的内容变化
12345
```

#### 3、版本回退

```javascript
// 现在->过去
git log                           // 查看历史记录  
git log --prettry=oneline         // 查看历史记录-简易版
git reset --hard HEAD^            // 回退到上一个版本
git reset --hard HEAD~10          // 回退到第前10个版本

// 过去->现在  
git reflog                        // 获得所有提交命令的版本号  
git reset --hard <commit id>      // 通过版本号回到现在  
123456789
```

#### 4、缓存区和暂存区

```javascript
git add file1 file2 file3         // 添加到缓存区
git add .                         // 添加全部修改文件 
git commit -m "name"              // 一次性提交多个文件
123
```

#### 5、撤销和删除文件

```javascript
// 文件内容有误，需要恢复到之前的版本：可以手动更改在commit，也可以回到HEAD^版本，本文介绍第三种方法

// - version1：没有加入到暂存区  
git status                        // 查看哪个文件被更改了
git checkout --filename           // 撤销这个文件的更改 

// - version2: 已经加入到暂存区  
git reset --hard HEAD^            // 先返回到上一版本（暂存区->工作区）
git checkout --filename           // 撤销这个文件的更改

rm filename                       // 从工作区删除filename  
git rm filename                   // 从版本库删除filename
git checkout -- filename          // 恢复删除的filename
12345678910111213
```

#### 6、远程仓库

```javascript
ssh-keygen -t rsa –C “youremail@example.com”    // 建立github和本地电脑的SSH Key链接  

// 本地->GitHub
git remote add origin 地址  		    // 关联一个GitHub
git push -u origin master          // 本地内容推送到GitHub（第一次用）
git push origin master             // 以后每次提交用

// GitHub->本地
git clone git地址
git pull origin master    		   // 拉取最新主分支代码
12345678910
```

#### 7、创建和合并分支

```javascript
git checkout -b feature1       // 创建并切换到feature1分支
git branch                     // 查看当前所有分支
git checkout master            // 切换到主分支  
git merge feature1             // 合并master和feature1分支:fast-mode模式
git merge --no-ff -m "merge with no-ff" <name>    // 合并分支，并且留下信息说明我在这里合并过 
git branch -d feature1         // 删除feature1分支

// 解决合并冲突
git log --graph --pretty=oneline --abbrev-commit   // 树状图查看分支情况
123456789
```

### 4、git常见问题解决方案

#### 1、代码推送

`git clone url`拉取代码后，`git branch -a`处于 `master` 分支，创建个人本地分支，`git checkout -b username`，
`git pull origin master`，保证个人本地分支代码与 `master` 分支代码相同，然后（创建远程分支，将本地个人分支代码与远程分支创建关联并推送到远程分支）
`git push origin username:username` 前面一个远程分支名称可以随便取的。

#### 2、代码改蹦

代码改蹦，对本地代码中修改的部分不做保存，具体：
1、`git fetch --all`
2、`git reset --hard origin/master`，这里的 `master` 可以是远程个人分支的分支名

#### 3、commit报错

当 `git commit` 的时候报错：`husky > pre-commit (node v12.13.0) Stashing changes...`
输入 `git commit -m ‘xxx’ --no-verify` 绕过了 `lint` 的检查即可

#### 4、拉取指定分支代码

在 `master` 分支拉取指定远程分支代码
`git fetch origin` 本地分支名：远程分支名称，然后切换个人分支 `git log` 即可
或者处在 `a` 分支，想啦b分支的代码，
`git checkout b`
`git pull origin b` 即可

#### 5、恢复代码

在 `git status` 时候发现有某个文件已经改动（还没有 `add` 和 `commit` ），但是现在不想改动，想还原，而且只需要还原这个文件，其他的改动继续提交
`git checkout -- @/aaa/bbb/xxx.vue `即恢复到修改之前的代码

#### 6、切换镜像

淘宝镜像： `npm install -g cnpm –registry=https://registry.npm.taobao.org`

#### 7、修改用户名

git修改用户名
`git config --system --unset credential.helper`
`git config --global credential.helper store`

#### 8、clone失败

```
$ git clone https://github.com/PanJiaChen/vue-element-admin.git`
`Cloning into 'vue-element-admin'...`
`fatal: unable to access 'https://github.com/PanJiaChen/vue-element-admin.git/':`
`OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443`
解决办法：将上面的 `https` 改为 `git` ,即 `git://github.com.......
```

#### 9、autocrlf报错

```
warning: LF will be replaced by CRLF in --->`
解决办法： `git config --global core.autocrlf false
```

#### 10、删除远端分支报错

```
git push origin --delete username` 报错
删除远程分支报错 `remote refs do not exist`
解决办法： `git fetch -p origin
```