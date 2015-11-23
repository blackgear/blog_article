title: Homebrew软件包的编写与提交
date: 2015-04-04 00:18:00
categories: network
tags: [homebrew, formula, debug]
description: 本文介绍了更新以及创建一个Homebrew里的软件包，并进行调试和发布的方法。
---

## 相关说明

目前国内已有很多关于如何使用Homebrew的文章，但是却很难找到一份关于如何编写Homebrew的软件包的介绍，Homebrew是一个基于git的软件包管理器，每个人都可以向Homebrew提交新的软件包或是提交已有软件包的更新。本文将用两个例子分别介绍如何提交已有软件包的更新以及如何提交新的软件包。

## 编写已有软件包的更新

Homebrew的软件包被称为formula，对应一个ruby脚本。我们以shadowsock-libev为例，介绍如何编写已有软件包的更新。首先在terminal中执行：

    $ brew edit shadowsocks-libev

brew将使用`$EDITOR`变量中指定的编辑器打开对应的ruby脚本，要更新一个软件包，我们只需要更改脚本的前三行：

    class ShadowsocksLibev < Formula
      homepage "https://github.com/shadowsocks/shadowsocks-libev"
      url "https://github.com/shadowsocks/shadowsocks-libev/archive/v2.1.4.tar.gz"
      sha256 "d4e665e375224ba1d4844b97e7263491ce07a60f08c9cb55c3128a6d3aad13e7"

这是shadowsocks-libev v2.1.4版本对应的脚本代码，脚本的第三行对应的就是这个软件包源代码的下载地址，修改第三行的url为新版本v2.2.0的软件包源代码下载地址，保存并执行如下命令：

    $ brew reinstall shadowsocks-libev

此时我们会获得这样的提示：

    Error: SHA256 mismatch
    Expected: d4e665e375224ba1d4844b97e7263491ce07a60f08c9cb55c3128a6d3aad13e7
    Actual: 49688f39649f0f61e323ddba8b02daa5dfe88bf2e051ed91181d266fe824df69

我们再次编辑软件包脚本，将第四行的SHA256值修改为新版本的`49688f39649f0f61e323ddba8b02daa5dfe88bf2e051ed91181d266fe824df69`即可。

重新执行命令测试安装shadowsocks-libev：

    $ brew reinstall shadowsocks-libev

现在没有错误出现了，我们的软件包更新的编写就完成了，接下来需要提交这个更新。

## 提交已有软件包的更新

首先访问[Homebrew的项目主页](https://github.com/Homebrew/Homebrew)，随后点击fork。本文已经假设你拥有一个Github帐号，并且拥有一些基本的git使用经验，如果你没有，还是多谷歌吧。

执行命令：

    $ brew update
    $ cd $(brew --repository)
    $ git checkout -b shadowsocks
    $ git add Library/Formula/shadowsocks-libev.rb
    $ git commit

我们首先将Homebrew更新到了最新版本，然后切换到了Homebrew的本地根目录，随后创建并切换到了一个名叫`shadowsocks`的branch，添加了新的脚本文件修改记录，最后进行commit。

Homebrew推荐的commit message style是非常简洁的：只需要软件包名加上版本号即可，比如：`shadowsocks-libev 2.2.0`。注意使用`foobar x.x.x`这样的格式，不要加入字符`v`或者`ver`等来表示版本号，直接使用数字就可。

将这个branch push到自己fork的Homebrew下：

    $ git push git@github.com:Github用户名/Homebrew.git shadowsocks

最后回到[Homebrew的项目主页](https://github.com/Homebrew/Homebrew)并提交一个pull request，title同样使用`shadowsocks-libev 2.2.0`即可。

我们只需要等待这个更新被接收即可。

最后，我们切换回master分支，并且删除掉shadowsocks分支。

    $ git checkout master
    $ git branch -D shadowsocks

## 创建全新的软件包

首先我们需要获取软件源代码包的地址，以cidrmerge为例，直接从sourceforge上下载时可以获得这个路径：

    http://iweb.dl.sourceforge.net/project/cidrmerge/cidrmerge/cidrmerge-1.5.3/cidrmerge-1.5.3.tar.gz

这个路径仅仅是sourceforge使用的镜像站点的地址，实际上我们应该使用这个地址以方便世界各地的用户从不同的镜像站下载源代码：

    https://downloads.sourceforge.net/project/cidrmerge/cidrmerge/cidrmerge-1.5.3/cidrmerge-1.5.3.tar.gz

执行命令：

    $ brew create https://downloads.sourceforge.net/project/cidrmerge/cidrmerge/cidrmerge-1.5.3/cidrmerge-1.5.3.tar.gz

Homebrew会按照模板创建一个默认的脚本：

    # Documentation: https://github.com/Homebrew/Homebrew/blob/master/share/doc/Homebrew/Formula-Cookbook.md
    #                /usr/local/Library/Contributions/example-formula.rb
    # PLEASE REMOVE ALL GENERATED COMMENTS BEFORE SUBMITTING YOUR PULL REQUEST!

    class Cidrmerge < Formula
      homepage ""
      url "http://iweb.dl.sourceforge.net/project/cidrmerge/cidrmerge/cidrmerge-1.5.3/cidrmerge-1.5.3.tar.gz"
      version "1.5.3"
      sha256 "21b36fc8004d4fc4edae71dfaf1209d3b7c8f8f282d1a582771c43522d84f088"

      # depends_on "cmake" => :build
      depends_on :x11 # if your formula requires any X11/XQuartz components

      def install
        # ENV.deparallelize  # if your formula fails when building in parallel

        # Remove unrecognized options if warned by configure
        system "./configure", "--disable-debug",
                              "--disable-dependency-tracking",
                              "--disable-silent-rules",
                              "--prefix=#{prefix}"
        # system "cmake", ".", *std_cmake_args
        system "make", "install" # if this fails, try separate make/make install steps
      end

      test do
        # `test do` will create, run in and delete a temporary directory.
        #
        # This test will fail and we won't accept that! It's enough to just replace
        # "false" with the main program this formula installs, but it'd be nice if you
        # were more thorough. Run the test with `brew test cidrmerge`. Options passed
        # to `brew install` such as `--HEAD` also need to be provided to `brew test`.
        #
        # The installed folder is not in the path, so use the entire path to any
        # executables being tested: `system "#{bin}/program", "do", "something"`.
        system "false"
      end
    end

这个小程序没有任何依赖项，只需要执行`make`就能完成编译，只需要保留编译完成得到的cidrmerge程序即可，所以只需要添加上项目主页的网址，并编写简单的安装脚本：

    class Cidrmerge < Formula
      homepage "http://cidrmerge.sourceforge.net"
      url "https://downloads.sourceforge.net/project/cidrmerge/cidrmerge/cidrmerge-1.5.3/cidrmerge-1.5.3.tar.gz"
      sha256 "21b36fc8004d4fc4edae71dfaf1209d3b7c8f8f282d1a582771c43522d84f088"

      def install
        system "make"
        bin.install "cidrmerge"
      end
    end

使用`system`表示执行后续的命令：

    system "make"

使用`bin.install`表示在bin目录下安装cidrmerge文件：

    bin.install "cidrmerge"

类似的还有：`sbin.install`、`etc.install`等诸多命令可以使用，具体请查阅[官方说明](https://github.com/Homebrew/Homebrew/blob/master/share/doc/Homebrew/Formula-Cookbook.md)。

最后加上测试脚本：

    test do
      input = <<-EOS.undent
        10.1.1.0/24
        10.1.1.1/32
        192.1.4.5/32
        192.1.4.4/32
      EOS
      assert_equal "10.1.1.0/24\n192.1.4.4/31\n", pipe_output("#{bin}/cidrmerge", input)
    end

这样，一个完整的安装包脚本就编写完成了。

## 提交全新的软件包

提交全新的软件包和提交软件包更新的方法类似，也是单独开一个branch、commit、push to remote，不过commit message需要使用`foobar 7.3 (new formula)`这样的形式。pull request的title也需要使用相同的形式。

通常你的软件包并不会一次过关，常常需要按照要求更改几次，在本地进行修改之后重新commit，然后push到你的Homebrew的fork的相同的分支下。此时可以随意填写commit message。比如cidrmerge就被提了[许多建议](https://github.com/Homebrew/Homebrew/pull/38332)。

当你的软件包通过审核之后，你的所有commits会被squash到第一个`foobar 7.3 (new formula)`的commit下，然后被merge到Homebrew项目，随后pull request被关闭。

随后你就会收到祝贺与感谢，比如：

> Thanks for the pull request! 🎉 Homebrew depends on contributions from community members like you and we're grateful for your support.
