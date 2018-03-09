About git
===

add ssh.pub to git,then you are not able to input your github andress and password
---
1.ssh-keygen -t rsa -C "yourEmail@example.com"
2.add .pub to github setting

command about git
---
1.init a repository
	* cd xxx
	* git init
2.add repository to remote server
	* git remote add origin "http://github.com/{USERNAME}/xxx.git"
3.clone repository from remote server
	* git clone -b {branch} "http://github.com/{USERNAME/xxx.git}"
4.add new file to cache
	* git add xxx
	* git add *
5.commit change to HEAD:after git add or modify the project
	* git commit -am "commit message"
6.submit 
 	* git push origin {branch}
7.about branch
	* git checkout xxx
	* -b create a branch
	* -d delete a branch
	* git merge {branch1} {branch2}
8.update local repository
	* git pull
9.TODO
