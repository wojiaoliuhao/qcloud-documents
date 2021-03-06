## 操作场景
通过添加 A 记录可将域名指向一个 IP 地址（外网地址），本文档指导您如何添加 A 记录。

## 操作步骤
1. 登录 [腾讯云云解析控制台](https://console.cloud.tencent.com/cns)。
2. 在 “域名解析列表” 中，选择需要进行 A 记录转发的域名，进入域名详情页面。
3. 单击【添加记录】，填写以下记录信息。
 ![](https://main.qcloudimg.com/raw/9b5acfbd4ff2e64e3d48bdd7d863e848.png)
 - 主机记录：选择子域名。例如添加  `www.123.com`  的解析时，您在 “主机记录” 处选择 “www” 即可。若是想添加 `123.com` 的解析，您在 “主机记录” 处选择 “@” 即可。
 - 记录类型：选择 “A”。
 - 线路类型：选择 “默认” 类型，否则会导致部分用户无法解析。
例如，您需要将联通用户指向 `8.8.8.8`，所有非联通用户都指向 `10.10.10.10`。您可以通过添加线路类型为默认、记录值为 `10.10.10.10` 和线路类型为联通、记录值为 `8.8.8.8` 的两条 A 记录来实现。
 - 记录值：只可以填写 IPv4 地址。
 - MX 优先级：不需要填写。
 - TTL：为缓存时间，数值越小，修改记录各地生效时间越快，默认为600秒。
4. 单击【保存】，完成添加。

>? 操作过程中如果出现问题，请您 [联系我们](https://cloud.tencent.com/document/product/302/33949)。


