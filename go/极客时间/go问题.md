

* go的环境变量如下：

![](/media/hpsyche/_dde_data/note/go/极客时间/pict/q-2.png)

其中GOPATH代表go的项目路径

>可以选择设置多个，即一个项目一个gopath，当较为麻烦，每次新建项目都需要配置
>我们采用设置一个gopath来下包含多个项目

Go的项目路径：	

![](/media/hpsyche/_dde_data/note/go/极客时间/pict/q-1.png)

其中bin代表生成的可执行文件；pkg是编译后生成的包的目标文件；src是每一个子目录，都是一个项目，如我这里创建了gitter_learn和goconvey两个项目。

* goconvey，在使用convey的webUI界面时，需要将bin下的convey可执行文件放到制定项目的根目录（或者想要测试的文件目录下）。



