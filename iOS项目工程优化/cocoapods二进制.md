# cocoapods-xlbuild

利用cocoapods，生成预编译静态库，提高编译速度的插件。支持编译使用静态库提高速度，调试直接使用源码，方便调试

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E8%83%8C%E6%99%AF)背景

随着项目的不断迭代，项目代码及依赖三方库和内部库越来越多，导致项目编译时间越来越长，浪费大量开发时间。 cocoapods-xlbuild插件将三方库打包为静态库，从而提高项目编译时间

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E6%8F%92%E5%85%A5)插入

$ gem install cocoapods-xlbuild

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E4%BD%BF%E7%94%A8)使用

修改 podfile 文件，加入以下代码

#### [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#1-%E4%BD%BF%E7%94%A8%E9%9D%99%E6%80%81%E5%BA%93%E7%BC%96%E8%AF%91)1. 使用静态库编译：

plugin 'cocoapods-xlbuild'
use_frameworks! :linkage => :static
use_static_binary!

使用动态库编译(动态库会拖累app使用时间，推荐使用静态库)：

plugin 'cocoapods-xlbuild'
use_frameworks!
use_dynamic_binary!

#### [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#2-%E5%A6%82%E6%9E%9C%E6%9F%90%E4%B8%AA%E5%BA%93%E4%B8%8D%E6%83%B3%E4%BD%BF%E7%94%A8%E9%A2%84%E7%BC%96%E8%AF%91%E5%8A%A0%E5%8F%82%E6%95%B0-binary--false)2. 如果某个库不想使用预编译加参数 :binary => false

pod 'AFNetworking', :binary => false

注意： 如果对某个库使用 `:binary => false` 则它的依赖库也不会预编译。 如果只想让当前库不参加预编译，依赖库参加预编译，可以将依赖库写在Podfile文件中 举个🌰： YTKNetwork、AFNetworking 都不参加预编译

pod 'YTKNetwork', :binary => false 

YTKNetwork不参加预编译，AFNetworking参与预编译

pod 'YTKNetwork', :binary => false 
pod 'AFNetworking'

#### [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#3-%E5%8F%AF%E4%BB%A5%E8%AE%BE%E7%BD%AE%E7%BC%96%E8%AF%91%E5%8F%82%E6%95%B0%E9%BB%98%E8%AE%A4%E4%B8%8D%E8%AE%BE%E7%BD%AE-%E4%BE%8B%E5%A6%82)3. 可以设置编译参数，默认不设置 例如：

set_custom_xcodebuild_options_for_prebuilt_frameworks :simulator => "ARCHS=$(ARCHS_STANDARD)"

#### [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#4-%E8%AE%BE%E7%BD%AE%E7%BC%96%E8%AF%91%E5%AE%8C%E6%88%90%E5%90%8E%E7%A7%BB%E9%99%A4%E6%BA%90%E7%A0%81%E9%BB%98%E8%AE%A4%E4%BF%9D%E5%AD%98)4. 设置编译完成后移除源码，默认保存

remove_source_code_for_prebuilt_frameworks!

#### [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#5-%E8%AE%BE%E7%BD%AEframeworks%E7%BC%93%E5%AD%98%E4%BB%93%E5%BA%93-install%E5%8A%A0%E9%80%9F-%E4%BE%8B%E5%A6%82)5. 设置Frameworks缓存仓库 install加速 例如:

set_local_frameworks_cache_path     '/Users/xxx/Desktop/CacheFrameworks'

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95)源码调试

不要设置 `remove_source_code_for_prebuilt_frameworks!` 选项，保留源码 源码将会放入pod工程 `SourceCode` 文件夹下，可以直接进行源码调试功能

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E6%B3%A8%E6%84%8F)注意

目前是直接将静态库引入至Pods中，注意将Pods添加到gitignore中，否则将会提交至git仓库中

如果工程里面一开始配置的是打包成动态库，后面再改成静态库需要将Pod目录删掉再重新执行pod install，否则就会导致某个pod版本更新了却未重新构建

## [](https://github.com/Jacky-LinPeng/cocoapods-xlbuild#%E5%8F%82%E8%80%83)