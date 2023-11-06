
背景：
国际化有第三方登录内容，Google，Facebook，Whatsapp 等。原先是第三方登录有个共有的基类
``` java
BaseThiredLogin
```
第三方登录分别实现。GoogleLogin FacebookLogin 。直接在UnifiyLogin里进行初始化。无需区分 客户端等等。

但是对于 华为手机 ，其登录所需要的 hwid 服务依赖HMS，不再Google的手机自带，同时 Google 登录的 play-service-auth 依赖GMS，也不在华为手机存在。

因此需要对华为手机适配 对于华为系统 使用华为登录。

改造：
对于GMS 和 HMS 抽出一个 MS LoginAdapter，屏蔽内部的实现细节，配置两端分别要支持的登录方式，GMS LoginAdapter（G F）/ HMS LoginAdapter（H F）
从登录库移除 原先第三方登录的具体实现。由业务库来直接实现。
方案两种：1. 根据flavor配置diff，在各自的业务代码中进行 MS LoginAdapter 的载入。
        2. 直接在dependencies上进行修改，封装 MS LoginAdapter 的实现为AAR，对于不同falvor进行引入，中业务侧通过SPI的方案 来加载对应的类实现，从而实现MS LoginAdapter的载入。

