# 背景
目前关于机器人开机自启动的设置大体上有下面几种方法：　　
```
    1、使用自启动的ros包upstart
    2、利用ubuntu自带的开机启动文件
    3. Linux 经典方式, 开机启动Service
    4. 简单暴力的 rc.local (问题较多的方法)
```
# ROS官方方法：
Robot_upstart:The robot_upstart package provides scripts which may be used to install and uninstall Ubuntu Linux upstart jobs which launch groups of roslaunch files.所以，这个方法可能仅支持Ubuntu Linux。它提供scripte来安装和卸载随机启动的ROS工作。

## 安装(kinetic-ROS下)
```
＄sudo apt-get install ros-kinetic-robot-upstart 
```
## 设置某个Package的launch文件为开机启动
假如有个Package：beacool_bringup, 它其中有个 .launch文件----beacool_bringup/launch/minimal.launch
```
$rosrun robot_upstart install beacool_bringup/launch/minimal.launch`
```
这里容易出以下几个问题：
```
    A. 找不到launch文件：
    A1：确保package_name/launch/目录的正确。
    A2：确保roscd package_name 可以进入。(说明：source ~/catkin_ws/devel/setup.bash 已经起效)
```
另一点要注意的是Ｌｏｇ：
```
    Preparing to install files to the following paths:
    /etc/init/beacool.conf
    /etc/ros/indigo/beacool.d/.installed_files
```
此时，Ubuntu启动时，这个Launch就会被调用。
## 其它关键信息
* service名
service名为：包名的一部分，在以上例子中，Package_Name: beacool_bringup.   所以service名为：beacool
* 停止开机启动
```
	rosrun robot_upstart uninstall service
	rosrun robot_upstart uninstall beacool
```
* service 启动和停止
```
	sudo service beacool start
	sudo service beacool stop
```
* 察看log 
```
sudo vi /var/log/upstart/beacool.log
```
网络上不少地方在描述这部分工作时：均不讲清楚 service名从何而来。带来不少误解。所以专门用红色标记service名.

# 使用Linux桌面系统对应开机启动程序：
* 1、点击Dash home打开Dash home。在检索窗里键入Startup application；也可以在终端输入：gnome-session-properties来启动。
"da"
* 2、显示出来的“Startup Applications”称为Startup Application Manager。点击打开Startup Application Manager,点击Add
* 3、“Name”可以是任意名称,给要启动的程序起个名字。当然是简单易懂明了的为好。
* 4、“Command”处记入启动电脑时要执行的命令或者是要启动的程序的所在,可以使用“Browse”这个按钮来设定要启动的程序的所在。
* 5、“Comment”处可记入一些说明。也可以不记入（当然“name”的地方也可以不记入。不过以后用起来很不方便）。
点击下面的“Add”按钮即可设定好想要在启动电脑时启动的程序了。
ROS程序启动思路很简单，就是运行一个终端程序，终端程序则运行一个脚本，这个脚本中设置ROS， 启动ROS应用程序。假如想启动的 .launch为： beacool_bringup 包内的minimal.launch，所以创建启动脚本如下：
```
	#! /bin/bash 
	source /opt/ros/indigo/setup.sh 
	 
	source /home/exbot/devel/setup.bash 
	roslaunch beacool_bringup minimal.launch
```
其次：在启动程序的命令列表中,使用：gnome-terminal -x　roslaunch beacool_bringup minimal.launch
则系统每次启动后，会开启一个终端窗口，并执行脚本中的launch。此方法在大部分ubuntu+ROS下有用。但在TK1 Ubuntu下未起作用，暂时不知怎么回事

# Linux开机启动Service
首先介绍背景知识：Linux启动时，可以启动一些Service。 Sam很早之前搞Linux机顶盒，Android机顶盒时。很多程序就是以Service形式启动的。
## 创建Service的方法
A: 在/etc/init.d/目录下，创建要启动的脚本。例如名为：beacool_rfkill 这个脚本用来启动 bluetooth.内容如下：
```
rfkill unblock bluetooth
```
B: 把脚本增加入Service：
```
	$ cd /etc/init.d
	$ sudo update-rc.d beacool_rfkill defaults 95
```
此时，会把脚本beacool_rfkill 加入到/etc/rcX.d/目录中。X:0-6, 分别表示不同的启动级别。3为字符界面启动，5为GUI启动。其它不关键。脚本名则在/etc/rc3.d/中变为：S95beacool_rfkill，命令中，最后的数字表示表示启动顺序。
C: 如果想要去掉此Service：
```
	$ cd /etc/init.d
	$ sudo update-rc.d -f beacool_rfkill remove
```
## ROS程序使用Service方式启动
为了查看脚本是否有效，可以查看log. Sam在脚本中作了如下动作：
```
roslaunch beacool_nav amcl_demo.launch >> /home/ubuntu/catkin_ws/src/beacool.log 2>&1
```
这样，就会把log 发送到指定目录。Log如下： 
```
/etc/rc2.d/S96dashgo_launch: 1: /etc/rc2.d/S96dashgo_launch: roslaunch: not found
```
这说明 ros没加入环境变量中。需要先把环境变量设置好。可参见后面的写法：
```
	/opt/ros/indigo/setup.sh 
	/home/ubuntu/catkin_ws/devel/setup.sh 
	roslaunch beacool_bringup minimal.launch
```
此办法有个缺点：如果有多个ROS工程要加入，例如：其中一个加为S98， 另一个加为S99。则S98会被执行。而S99并未执行。虽然也可以后台执行，但会导致前一个退出。所以如果一定要采用此方法：可以把多个Launch合并。(并不建议)


# 简单暴力的 rc.local (问题较多的方法)
/etc/rcX.d/中，都包含了/etc/init.d/rc.local.  通常都是最后一刻加入的，例如：S99rc.local，而它又引用了 /etc/rc.local，所以，可以简单的在/etc/rc.local 的exit 0 之前加入调用。例如：想要加入一个ROS项目：beacool_bringup 的minimal.launch.

A: 先创建一个script:
```
	./opt/ros/indigo/setup.sh 
	./home/ubuntu/catkin_ws/devel/setup.sh 
	roslaunch beacool_bringup minimal.launch 
```
请注意：此时并没有使用source /opt/ros/indigo/setup.bash，而是采用 .  /opt/ros/indigo/setup.sh ，是因为此时bash还未启动，而source是bash内部命令。此时找不到source.
B: 修改其权限：
```
sudo chmod 777 beacool_launch
```
C: 加入rc.local:
```
/etc/init.d/beacool_launch
```
此办法有个和方法3类似的缺点：
* 加入后，如果ROS 程序不退出，则阻塞在这里。这意味着如果要启动多个launch. 则下一个不会被执行。
* 当然有朋友会说，可以后台执行嘛。/etc/init.d/beacool_launch & 
但如果后面再启一个launch, 前面beacool_launch所启动的ROS项目会退出。所以如果一定要采用此方法：可以把多个Launch合并。(并不建议)设置了程序自启动之后，需要把Ubuntu的开机密码取消掉，这样就不需要人为登录，设置方法参考如下：
https://jingyan.baidu.com/article/0320e2c1d4da0b1b87507b31.html
