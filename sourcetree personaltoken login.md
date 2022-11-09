使用sourcetree更新GitHub项目时报错：`Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.`

因为github更新策略，只可用personal access token登录，sourcetree 修改方法：

在sourcetree中进入项目，点击【远程仓库】修改仓库地址：

![](https://www.crazyming.com/wp-content/uploads/2021/09/GitHub%E2%80%94%E2%80%94sourcetree-token.png)

格式为：`https://<your_token>@github.com/<USERNAME>/<REPO>.git`