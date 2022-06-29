# git命令

#### remote远程仓库origin添加错了，删除远程仓库

远程仓库名称是origin

```shell
git remote rm origin
```

#### 显示所有远程仓库：

```shell
git remote -v
```

#### 添加远程版本库：

```shell
git remote add origin git@github.com:YJL857/ITfield.git
```

#### 查看git状态

```shell
git status
```

#### add命令

```shell
#添加单个文件夹
git add ULServer/ 
#添加所有 文件
git add .
```

#### commit命令

```shell
git commit -m "提交项目代码"
```

#### push命令

以下命令将本地的 master 分支推送到 origin 主机的 master 分支。

```shell
git push origin master
```

#### 本地文件上传到git（第一次）

- 将远程仓库的文件拉取到本地，例如文件夹名称叫  ITfield

```shell
git clone git@github.com:YJL857/ITfield.git
```

- 将文件复制到  ITfield  文件夹中

- 进入   ITfield  文件夹

- ```shell
  git add .
  git commit -m "提交项目代码" 
  git push origin master
  ```