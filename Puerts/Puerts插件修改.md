在自己的UE项目中，Puerts一般都是在
/Plugin/Puerts
而官方仓库中仓库结构包含了unity和ue的
/unity 
/unreal/Puerts

我们项目中要对Puerts做一些修改，而且还要保持与官方的更新同步

采用 git subtree + split 子目录
# 初始化Puerts插件
自己Fork一个Puerts仓库

1. 将Fork仓库clone下来，把 Puerts 的子目录拆出来
```bash
git clone https://github.com/zopeplone/puerts.git
cd puerts
git subtree split --prefix=unreal/Puerts -b puerts-unreal
```
- 得到一个新分支 `puerts-unreal`
- 这个分支只包含 `/unreal/Puerts`
- 历史干净保留
`puerts-unreal` 分支 只存官方代码（子目录提取结果）绝对不要改
2. 把这个“子仓库”加到项目里
```bash
git subtree add --prefix=Plugins/Puerts https://github.com/zopeplone/puerts.git puerts-unreal --squash
```
所有对Puerts的修改都是在项目中的
>需要确保Plugins/Puerts 是空的，否则报错 fatal: prefix 'Plugins/Puerts' already exists.
# 如果自己有修改
直接commit即可

# 同步官方更新

当官方 Puerts 更新：
1. Fork仓库同步官方main分支
2. 在仓库中rejoin puerts-unreal分支
```bash
git subtree split --prefix=unreal/Puerts --rejoin -b puerts-unreal
```
3. 同步到项目中
```bash
git subtree pull --prefix=Plugins/Puerts https://github.com/zopeplone/puerts.git puerts-unreal --squash
```
可以自动merge