# git
</br>

## git命令

| git命令 |  作用  |
|:--------|:------|
| git remote add origin git@github.com:Sestar/demo.git | 连接上gitHub, Sestar账号, demo仓库 |
| git remote set-url origin git@github.com:Sestar/demo.git | 修改gitHub地址 |
| | |
| git add 文件 | 将文件放入本地暂缓区 |
| git commit -m "自定义提交信息" | 提交 |
| git log  |  可以查看提交历史，以便确定要回退到哪个版本 |
| git reflog  |   查看命令历史，以便确定要回到未来的哪个版本 |
| | |
| git config user.name | 查看git登录账号
| git config user.email | 查看git登录邮箱
| | |
| git reset --mixed  |  默认选项。缓存区和你指定的提交同步，但工作目录不受影响 |
| git reset --soft  |  缓存区和工作目录都不会被改变 |
| git reset --hard  |  缓存区和工作目录都同步到你指定的提交 |
| git reset --mixed HEAD  |  将你当前的改动从缓存区中移除，但是这些改动还留在工作目录中 |
| | |
| git status | 查看暂缓区(git的缓存)修改内容 |
| git reset HEAD | 重回上次更新版本 |
| git checkout --文件 |  将工作区(代码)修改记录删除 |
| | |
| git rm 文件 | 删除git文件(缓存区) |
| | |
| git branch -a | 查看远程所有分支 |
| git branch | 查看本地所有分支 |
| git branch test | 创建'test'分支 |
| git checkout test | 切换到'test'分支 |
| | |
| git remote -v | 查看远程仓库地址 |


</br>

## 撤销修改

### commit 但是没有 push

```text
git reset --soft|--mixed|--hard <commit_id>

--mixed  会保留源码,只是将git commit和index 信息回退到了某个版本.
--soft   保留源码,只回退到commit信息到某个版本.不涉及index的回退,如果还需要提交,直接commit即可.
--hard   源码也会回退到某个版本,commit和index 都会回退到某个版本.(注意,这种方式是改变本地代码仓库源码)
```

### 已经push

```text
git revert <commit_id>

直接回退版本
```

</br>

## revert, reset, checkout区别

  * revert相比git reset，它不会改变现在的提交历史。因此，git revert可以用在公共分支上，git reset应该用在私有分支上。  
  * git revert当作撤销已经提交的更改，而git reset HEAD用来撤销没有提交的更改。  
  * revert对应的是本地仓库所有修改, reset对应的是暂缓区的所有修改, checkout对应的是某个文件的修改  

</br>

| 命令				|		作用域			|		  常用情景 |
|:---------------|:-----------------:|:---------------|
| git reset			|		提交层面		|	在私有分支上舍弃一些没有提交的更改
| git reset			|		文件层面		| 	将文件从缓存区中移除
| git checkout	| 	   提交层面	    | 		切换分支或查看旧版本
| git checkout	| 	    文件层面    	| 		舍弃工作目录中的更改
| git revert		| 	    提交层面	    | 		在公共分支上回滚更改
| git revert		| 	    文件层面	    | 			（然而并没有）


</br>

## 连接不上远程仓库

```text
出现fatal: Could not read from remote repository
git remote set-url origin 远程地址
还是不行, 重新生成ssh key
```

## 生成ssh文件

```text
Git Base 运行 $ ssh-keygen -t rsa -C "346825377@qq.com"
之后会让你输入三个密码, 全部填空即可
在 C:\Users\Sestar\.ssh 文件夹下会生成 id_rsa 和 id_rsa.pub 文件
在 Git Hub 的个人 ssh 设置中, title自定义, key填写id_rsa.pub的内容即可
```

## 删除文件Git关联

```text
直接删除 .git 文件夹(隐藏文件)
```

## unpopulated submodule

```text
我出现这个问题是因为在 当前目录（以下写：./）下建立了一个仓库：A 
而 ./ 下有一文件夹 命名为“A”，A/ 有之前建立的仓库，我在 ./ 下add commit push 后发现远程仓库内并没有A/的内容，  
于是我在 A/ 下执行 ”git add .” 提示：“in unpopulated submodule ‘A’ ”（翻译为”在一个无人居住的子模块“，感觉意思是说位于子模块下，
无法 add 0.0） 
解决方法是：

    1.删除 A/ 的.git 文件夹
    2.在 ./ 下输入”git rm -r –cached A/“ //谨记：是 A/ ，意为A目录下
    3.在 ./ 下输入”git add A”
    4.git commit -m “”
    5.git push origin master

我感觉主要是第二步操作：“git –cached < file >” 使名为file的文件不再接受版本控制，所以就没有了所谓的子模块的冲突了。
```

## GitHub
<br />

### 删除仓库
<br />

#### 文字描述

> 1.进入要删除的仓库， 选择Setting标签
> 2.直达页面最下方, 'Danger Zone'模块中有 Delete this repository 选项，点击即可

<br />

#### 图片示例

> 进入Setting标签

<img width="80%" src="/note/_v_images/code_sofa/git/删除仓库-1.png" />

> 删除仓库按钮

<img width="80%" src="/note/_v_images/code_sofa/git/删除仓库-2.png" />


<br />