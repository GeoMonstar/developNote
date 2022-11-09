cocopods的安装攻略】gem,Homebrew

Homebrew:是Mac OSX上的软件包管理工具，能在Mac中方便的安装软件或者卸载软件。相当于Linux里的yum、apt-get等软件管理工具。homebrew
gem:和brew不同，brew用于操作系统层面上的软件包的安装，而gem只是管理ruby软件
npm:是node.js界的程序/模块管理工具，也就是说npm只管理那些服务于JavaScript社区的程序。而且跨平台，windows和osx，以及其他unix like操作系统都可以用。npm



更换rvm版本
rvm管理ruby的版本的工具

// 安装
rvm implode 

rvm list   //列表 
rvm remove2.3
rvm install 2.7.2   //安装

// 使用
rvm --create ruby-2.7.2
rvm 2.7.2 --default
rvm use ruby-2.7.2

// gem异常
rvm pkg install openssl
rvm reinstall ruby-2.1 --with-openssl-dir=$rvm_path/usr

//移除
sudo gem uninstall cocoapods 

//组件
gem list --local | grep cocoapods
sudo gem uninstall 

///安装
gem install cocoapods


gem
gem里面有源，source。可以安装cocoapods

更新gem
sudo gem update —system
ERROR: While executing gem … (Errno::EPERM) Operation not permitted @ rb_sysopen - /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/gem
sudo gem update -n /usr/local/bin —system

更换源
gem sources —remove https://ruby.taobao.org/
gem sources -a https://gems.ruby-china.org

安装CocoaPods
sudo gem install cocoapods
sudo gem install -n /usr/local/bin cocoapods

更新索引仓库
pod setup

参考



删除cocopods
如果之前用gem安装过，或者别的方式安装过cocopods，可以删除

卸载命令
sudo gem uninstall cocoapods
报错
You don't have write permission for the /usr/bid directory

sudo gem install cocoa pods -n /usr/local/bin
查看相关的东西
///命令
gem list --local | grep cocoapods

cocoapods-core (0.39.0)
cocoapods-downloader (0.9.3)
cocoapods-plugins (0.4.2)
cocoapods-search (0.1.0)
cocoapods-stats (0.6.2)
cocoapods-trunk (0.6.4)
cocoapods-try (0.5.1)
// 逐个删除
sudo gem uninstall cocoapods-core


brew（推荐）
macOS 10.15 cocoapods gem安装报错，使用brew安装

安装前的准备
使用brew安装，也要先设置ruby的源

安装
方式一（常规安装）：

# /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
 方式二（换国内的镜像）：curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused（会删除之前的缓存）

 /bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
install cocoapods
# brew install cocoapods
cocoapods1.10.0安装成功以后，需要链接，链接成功即是最新版cocoapods

# brew link cocoapods
# brew link --overwrite cocoapods


cocopods（最终）
cocopods和上面用的工具brew，gem类似，都是包管理工具，cocopods也是根据源来下载资源。
目前都是新版的cocopods（1.8.0 版本的正式发布后，CDN 被作为了spec的默认来源），都使用的是在工程中添加source。

查看repo
pod repo
使用coco的源
#在podFile添加
source 'https://github.com/CocoaPods/Specs.git'
source 'https://github.com/Artsy/Specs.git'
#cdn
source 'https://cdn.cocoapods.org/'
使用清华的源
详细
#对于旧版的 CocoaPods 可以使用如下方法使用 tuna 的镜像：
$ pod repo remove master
$ pod repo add master https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git
$ pod repo update

#新版的 CocoaPods 不允许用pod repo add直接添加master库了，但是依然可以：
$ cd ~/.cocoapods/repos 
$ pod repo remove master
$ git clone https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git master

#使用1.8，CocoaPods不再需要克隆现在巨大的主规格repo才能运行，用户几乎可以立即将他们的项目与CocoaPods集成.
#最后进入自己的工程，在自己工程的podFile第一行加上：
source 'https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git




pod 速度慢建议倒入clash终端命令

##mac公钥
ssh-keygen -t rsa

然后一路回车

cat ~/.ssh/id_rsa.pub