F7:单步进入，可以用鼠标左键点击某个方法进入
F8:单步跳过
F9:跳过调试
项目忽略文件:Editor>File Types>Ignored Files and Folders
大小写转化: Shift+Ctrl+U
查看类的继承关系：Ctrl+Alt+U，可以看到继承了哪些类
查看类的被继承关系：Ctrl+H，可以看到类的子类
查看接口中声明方法的定义:Ctrl+Alt+B
maven配置自动下载依赖源码:File>Setting>Build,...>Maven>Importing，勾选Automatically download中的三个选项，然后在项目的pom.xml上reload项目就会下载依赖的源码
idea取消屏幕中间竖线:File>Editor>General>Apperance>Show hard wrap...
根据类名找到引入依赖的pom:先找到类所在的包以及所在的pom模块，在pom模块上Show Dependenices就能看到依赖视图，然后Ctrl+F搜索包名，点击包就能找到pom.xml。