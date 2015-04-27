#GitLab的安装方式
GitLab的两种安装方法:

* 编译安装
	* 优点：可定制性强。数据库既可以选择MySQL,也可以选择PostgreSQL;服务器既可以选择Apache，也可以选择Nginx。
	* 缺点：国外的源不稳定，被墙时，依赖软件包难以下载。配置流程繁琐、复杂，容易出现各种各样的问题。依赖关系多，不容易管理，卸载GitLab相对麻烦。
* 通过rpm包安装
	* 优点：安装过程简单，安装速度快。采用rpm包安装方式，安装的软件包便于管理。
	* 缺点：数据库默认采用PostgreSQL，服务器默认采用Nginx，不容易定制。

由于公司只配备了一台阿里云服务器，并且没有分配任何的域名。该服务器上需要运行版本控制软件、bug管理软件、知识库等多套程序，只能采用ip的方式访问。原先采用GitLab+Apache+MySQL编译安装的方式，并且将GitLab配置为可通过`xxx.xx.xxx.xx/gitlab`的形式访问，由于bug管理软件（禅道）也运行于Apache之上，两套软件之间彼此有互斥的影响，找不到解决方法。同时，GitLab的注册需要邮箱验证，由于网上提供的配置方法都是基于域名的，在阿里云上多次进行配置都无法正常使用。

因此，只能放弃编译安装的方式，而采取rpm包的方式重新进行安装。

#安装GitLab CE Omnibus包
1. 在linux终端下，使用`cat /etc/issue`命令查询当前系统的发行版本，查询到阿里云所安装的linux版本为CentOS release 6.6 (Final)。

2. 进入[gitlab官方网站](https://about.gitlab.com/downloads/),选择对应的操作系统——CentOS 6 (and RedHat/Oracle/Scientific Linux 6),按照官方的提示进行安装：

	1. 安装配置必要的依赖
		
		在Centos 6 和 7 中，以下的命令将会打开HTTP和SSH在系统防火墙中的可访问权限。
		
	   ```bash
		sudo yum install openssh-server
		
		sudo yum install postfix
		
		sudo yum install cronie
		
		sudo service postfix start
		
		sudo chkconfig postfix on
		
		sudo lokkit -s http -s ssh
		
		```

	2. 下载Omnibus package包并安装
		
		```bash
		curl -O https://downloads-packages.s3.amazonaws.com/centos-6.6/gitlab-ce-7.10.0~omnibus.2-1.x86_64.rpm
		sudo rpm -i gitlab-ce-7.10.0~omnibus.2-1.x86_64.rpm
		```
			Note:由于amazonaws的服务器被墙，下载这个包时可能需要翻墙下载。
	3. 配置并启动GitLab
	   打开`/etc/gitlab/gitlab.rb`,将`external_url = 'http://git.example.com'`修改为自己的IP地址：`http://xxx.xx.xxx.xx`,，然后执行下面的命令，对GitLab进行编译。
		
		```bash
		sudo gitlab-ctl reconfigure
		```
		
	4. 登录GitLab
		
		```
		Username: root 
		Password: 5iveL!fe
		```
		
# 配置GitLab的默认发信邮箱
1. GitLab中使用`postfix`进行邮件发送。因此，可以卸载系统中自带的`sendmail`。
使用`yum list installed`查看系统中是否存在`sendmail`，若存在，则使用`yum remove sendmail`指令进行卸载。
2. 测试系统是否可以正常发送邮件。
	```bash
	echo "Test mail from postfix" | mail -s "Test Postfix" xxx@xxx.com
	```
	
		注：上面的xxx@xxx.com为你希望收到邮件的邮箱地址。
	当邮箱收到系统发送来的邮件时，将系统的地址复制下来，如：`root@iZ23syflhhzZ.localdomain`,打开`/etc/gitlab/gitlab.rb`,将
	
	```
	# gitlab_rails['gitlab_email_from'] = 'gitlab@example.com' 
   ```
	
	修改为
	
	``` 
	gitlab_rails['gitlab_email_from'] = 'root@iZ23syflhhzZ.localdomain' 
	```
	保存后，执行`sudo gitlab-ctl reconfigure`重新编译GitLab。如果邮箱的过滤功能较强，请添加系统的发件地址到邮箱的白名单中，防止邮件被过滤。

		Note:系统中邮件发送的日志可通过`tail /var/log/maillog`命令进行查看。
 
 
# 安装过程中出现的问题
1. 在浏览器中访问GitLab出现`502`错误

	原因：内存不足。
	
	解决办法：检查系统的虚拟内存是否随机启动了，如果系统无虚拟内存，则增加虚拟内存，再重新启动系统。

2. `80`端口冲突

	原因：Nginx默认使用了`80`端口。
	
	解决办法：为了使Nginx与Apache能够共存，并且为了简化GitLab的URL地址，Nginx端口保持不变，修改Apache的端口为4040。这样就可以直接用使用ip访问Gitlab。而禅道则可以使用`4040`端口进行访问，像这样：`xxx.xx.xxx.xx:4040/zentao`。具体修改的地方在`/etc/httpd/conf/httpd.conf`这个文件中，找到`Listen 80`这一句并将之注释掉，在底下添加一句`Listen 4040`，保存后执行`service httpd restart`重启apache服务即可。
	
	```
	#Listen 80 
	Listen 4040 
	```
3. `8080`端口冲突
  
	原因：由于unicorn默认使用的是`8080`端口。
	
	解决办法：打开`/etc/gitlab/gitlab.rb`,打开`# unicorn['port'] = 8080 `的注释，将`8080`修改为`9090`，保存后运行`sudo gitlab-ctl reconfigure`即可。
	
4. STMP设置

	配置无效，暂时不知道原因。
	
5. GitLab头像无法正常显示
	原因：gravatar被墙
	解决办法：
	编辑 `/etc/gitlab/gitlab.rb`，将
	
	```
	#gitlab_rails['gravatar_plain_url'] = 'http://gravatar.duoshuo.com/avatar/%{hash}?s=%{size}&d=identicon'
	```
	
	修改为：
	
	```
	gitlab_rails['gravatar_plain_url'] = 'http://gravatar.duoshuo.com/avatar/%{hash}?s=%{size}&d=identicon'
	```
	然后在命令行执行：
	
	```bash
	sudo gitlab-ctl reconfigure 
	sudo gitlab-rake cache:clear RAILS_ENV=production
	```
	

# 参考资料：

[GitLab 6.1 使用postfix发送email](http://www.07net01.com/program/641580.html)

[Configure GitLab Omnibus installation alongside with Apache](http://devonoid.net/configure-gitlab-omnibus-installation-alongside-with-apache/)

[解决Gitlab的Gravatar头像无法显示的问题](http://my.oschina.net/anylain/blog/355797)

[How To Set Up GitLab As Your Very Own Private GitHub Clone](https://www.digitalocean.com/community/tutorials/how-to-set-up-gitlab-as-your-very-own-private-github-clone)
	