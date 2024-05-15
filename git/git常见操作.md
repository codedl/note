+ 忽略文件夹

编辑.gitignore文件，设置要忽略的文件夹
```
### ZK ###
data1
data2
data3
version-2
```
再清除缓存
```git
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```

+ 拉取远程分支覆盖本地分支
```git
git reset --hard origin/master
git push -f
```

+ 分支
```git
git checkout -b new //新建并切换
git branch new & git checkout new // 新建并切换
git checkout master //切换
git merge new //合并
git branch -d new //删除
```