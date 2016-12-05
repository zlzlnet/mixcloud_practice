# mixcloud_practice

## 混合云实践
0.1.3

本文主要记录一些混合云搭建的实践，包括zStack部署、虚拟机生命周期管理相关、镜像存储相关、网络相关等

## 主要修改历史
* 0.1.3 ：2016-12-5
	* 添加OpenStack controller节点环境搭建简介
	* 添加CentOS调整root目录分区大小的方法
* 0.1.2 ：2016-12-1
	* 添加OpenStack环境搭建章节简介 
* 0.1.1 ：2016-11-18
	* 添加虚拟机标准镜像制作方法
* 0.1.0 ：2016-11-17
	* 添加基本内容
	* 添加qcow2镜像扩容方法



## 参加步骤

* 在 GitHub 上 fork 到自己的仓库，如```user/mixcloud_practice```，然后 clone 到本地，并设置用户信息。

```
$ git clone git@github.com:user/mixcloud_practice.git
$ cd mixcloud_practice
$ git config user.name "yourname"
$ git config user.email "your email"
```
* 修改代码后提交，并推送到自己的仓库。

```
$ #do some change on the content
$ git commit -am "Fix issue #1: change helo to hello"
$ git push
```

* 在 GitHub 网站上提交 pull request。

* 定期使用项目仓库内容更新自己仓库内容。

```
$ git remote add upstream https://github.com/zlzlnet/mixcloud_practice.git
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
$ git push -f origin master
```
