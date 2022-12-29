# git remove big size files



## 服务端限制

### gitlab

1. 点击 `Menu`->`Settings`->`General`->`Account Limit`
1. 修改`Maxiumum attachment size` & `Maxiumum push size` 

## 查找大文件

查找排名前五的pack记录：

```bash
git verify-pack -v .git/objects/pack/pack-*.idx | sort -k 3 -g | tail -5
```
最后一条就是最大的一条记录，4cc1f9dcef1004355d2a595d45808e99f100dc4d 是它的 id。


找出该记录对应的文件：

```bash
$ git rev-list --objects --all | grep 4cc1f9dcef1004355d2a595d45808e99f100dc4d
4cc1f9dcef1004355d2a595d45808e99f100dc4d dist.zip
```
## 查看git commit更改的文件列表

查看git commit更改的文件列表, 重定向到commit.log. 在log文件中检索查看大文件对应的commit id。
```bash
git log --name-only > commit.log
```

## 彻底从git中清除

### 1.  移除git历史的blob文件
   
remove the blob file from our git history by rewriting the tree and its content.

the `rm` option removes the file from the tree. Additionally, the `-f`  option prevents the command from failing if the file is absent from other committed directories in our project. Without the `-f` option, the command may fail when we have more than one directory in our project.

```bash
$  git filter-branch --tree-filter 'rm -f dist.rar dist.zip' HEAD
```

### 2. git log移除被删文件

Our git log still contains the reference to the deleted file. We can delete the reference by updating our repo

The `-d` option deletes the named ref after verifying it still contains old values.

```bash
 git update-ref -d refs/original/refs/heads/${your_branch_name}
```

### 3. 移除过期日志项

We need to record that our reference changed in the repository:

```bash
git reflog expire --expire=now --all
```
The expire subcommand prunes older reference log entries.

### 4.  git gc

we need to clean up and optimize our repo:

```bash
git gc --prune=now
```
The `–prune=now` option prunes loose objects regardless of their age.

### 5. 强制推送

```bash
git push origin develop -f
```

## another way 


```bash
git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -5 | awk '{print$1}')"

```