不习惯使用SceneDelegate可以将其删除，按原来的方式进行项目开发
删除操作：
1、SceneDelegate文件删除
2、AppDelegate文件中函数application(_:configurationForConnecting:options:)和application(_:didDiscardSceneSessions:)删除
3、Info.plist文件中Application Scene Manifest删除
4、AppDelegate .h中加入@property** (**strong**, **nonatomic**) UIWindow *window 属性
