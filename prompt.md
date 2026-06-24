## 绕过 GLM 限制的 prompt（每次升级 hermes 后需要重新执行下面提示）：
拉取 https://github.com/Samge0/hermes-provider-api-workarounds.git 仓库到 hermes 的 skills 中并根据该 skill 经验配置绕过 GLM 的 429 限制。参考教程：https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded  

修改完毕后需要测试，测试通过后自行构建命令重启hermes-web-ui (不要留有僵尸旧进程，彻底关闭 hermes相关进程跟 hermes-web-ui 进程后重启 hermes-web-ui，批命令运行，不然会执行了关闭当前会话进程而没有执行重启命令)


## 如果执行一次没生效，可再提示一次：
似乎没有真正完成修改跟绕过，还是提示：Error: HTTP 429: 该模型当前访问量过大，请您稍后再试


## 如果还是不行，需要自己手动重启 hermes 等服务：
hermes-web-ui

hermes-web-ui stop

hermes-web-ui
