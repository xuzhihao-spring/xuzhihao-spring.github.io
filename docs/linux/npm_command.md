# npm命令

```bash
sudo curl -sL -o /etc/yum.repos.d/khara-nodejs.repo \ https://copr.fedoraproject.org/coprs/khara/nodejs/repo/epel-7/khara-nodejs-epel-7.repo
$ sudo yum install -y nodejs nodejs-npm #yum安装npm和nodejs

npm -v #查看npm安装的版本
npm install --registry=https://registry.npm.taobao.org #指定仓库地址
npm install moduleName #安装node模块
npm install moduleName@1.0.0 #安装node模块特定版本
npm install -g moduleName #全局安装命令
npm set global=true #设定全局安装模式
npm get global #查看当前使用的安装模式
npm install –save #将模块写入dependencies节点（生产环境）
npm install –save-dev #将模块写入devDependencies节点（开发环境）
npm outdated #检查包是否已经过时
npm update moduleName #更新node模块
npm uninstall moudleName #卸载node模块
npm init #会引导你创建一个package.json文件，包括名称、版本、作者这些信息等
npm root #查看当前包的安装路径
npm root -g #查看全局的包的安装路径
npm list #查看当前目录下已安装的node包
npm list parseable=true #以目录的形式来展现当前安装的所有node包
```