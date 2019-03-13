# 开始



[部署Node.js项目](https://help.aliyun.com/document_detail/50775.html)

[ linux下面mongodb安装及设置后台运行的方法](https://www.haorooms.com/post/linux_mongo_backupgo)

***

#### 项目

Vue2-mamager

1. cd opt/node/vue2-manage 
2. git pull origin master 
3. npm install 
4. npm run dev 

Node-elm

1. cd opt/node/node-elm
2. git pull origin master 
3. npm install 
4. 启用mongon
   1. cd  /usr/local/mongodb/bin
   2. ./mongod --fork --dbpath=/data/db --logpath=/usr/local/mongodb/logs --logappend
5. npm start 

#### 常用命令

1. ln -s /root/node-v6.9.5-linux-x64/bin/node /usr/local/bin/node  软链 

2. ps -A  显示所有进程信息 

3. ./mongod --fork --dbpath=/data/db --logpath=/usr/local/mongodb/logs --logappend

4. 关闭  mongod

   * ~~~shell
     $ ./mongod
     
     > use admin
     > db.shutdownServer()
     ~~~

   * 

***

##### 环境

1. python3
   1. wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
   2. mkdir -p /usr/local/python3
   3. tar -zxvf Python-3.6.5.tgz
   4. cd Python-3.6.5
   5. ./configure --prefix=/usr/local/python3
   6. make
   7. make install
   8. ln -s /usr/local/python3/bin/python3 /usr/bin/python3
   9. ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3