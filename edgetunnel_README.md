# Edge Tunnel

## toc

## 简介

基于3K (现改名为[6K](https://github.com/6Kmfi6HP/EDtunnel)) 的 EDTunnel 24年的代码修改，并结合 [zizifn](https://github.com/zizifn/edgetunnel) 大佬更新后的代码

- 将访问过的由 Cloudflare 托管的域名标记到 KV namespace，数据对应 KV key `cfhost` 及源文件 `src/cfhost.json`

- 如果请求的远端域名/IP在标记域名列表 cfhost 或 Cloudflare cidr ip 范围内，直接连接反向代理（而非先直连远端失败 再重连反向代理）。通过上述对标记域名的记忆(存储) 故得以实现

- 每日对反向代理IP进行连通性测试与过滤，数据对应 KV key `proxys` 及源文件 `src/proxys.json`

- 区分可以访问 openai、twitter(X) 的反向代理 IP

- GitHub action 工作流每日测试/过滤 持续集成上述数据，构建到 _worker.js，部署 worker / page

## 部署 worker/page

登录 cloudflare dash https://dash.cloudflare.com

登录 Github，fork 本仓库 https://github.com/geeksnail/edgetunnel ，导航到你fork来的仓库页的 'Settings'，左侧 'Secrets and variables' 下拉 'Actions'

**仓库密钥 Repository Secrets:** 

![image-20240627050801000](https://s2.loli.net/2024/06/27/q3IplbGBuFjLUMs.png)

- **`CF_ACCOUNT_ID`**，必填，登录 cloudflare 后地址栏 `https://dash.cloudflare.com/<ACCOUNT_ID>` 的后缀长串

- **`CF_API_TOKEN`**，必填，在页面 https://dash.cloudflare.com/profile/api-tokens - 创建 Token - Create Custom Token，如图:
  
  - Token name: 根据权限随意起
  - 设置 Account 3 个更改权限：Cloudflare Pages - Edit，Workers KV Storage - Edit，Workers Scripts - Edit
  - 其它权限不设置不影响
  
  创建后一定记得备份 token！！！
  
  
  ![image-20240627045709647](https://s2.loli.net/2024/06/27/lLDx9Se4VsRaF6H.png)
  
- **`CF_NAMESPACE_ID`**，可选，若未在下面的 Variables 中设置，则必须在此处设置。在 cloudflare dash 页左侧导航 - Storage & Databases - KV 页或地址栏输入  `https://dash.cloudflare.com/<ACCOUNT_ID>/workers/kv/namespaces`，Create a namespace，名字随意，会生成一个 ID。

  ![image-20240627045057635](https://s2.loli.net/2024/06/27/83ohHpe2ESvAUFb.png)

- `CF_WORKER_UUID`，可选，对应 _worker.js `env.uuid` 变量，也可在 cloudflare dash 后台部署好的 worker 页设置（如图）

- **`CF_PAGE_UUID`**，可选，对应 _worker.js `env.uuid` 变量，也可在 cloudflare dash 后台部署好的 page 页设置（如图）

  UUID生成：对于 Windows，按 "win + r", 输入命令 `Powershell -NoExit -Command "[guid]::NewGuid()"`；
  
  （可选）若安装了 V2rayN，点击"添加VLESS配置"，"UUID(id)" 右侧点击 "generate(生成)"
  
  ![image-20240627194432472](https://s2.loli.net/2024/06/27/PndJHLViTQXuD9h.png)

**仓库变量 Repository Variables:** 

![image-20240627050902767](https://s2.loli.net/2024/06/27/Ja9sZyTDwfpIcml.png)

- `CF_WORKER_NAME`，可选， `CF_WORKER_NAME` 和 `CF_PAGE_NAME` 至少填一个
- **`CF_PAGE_NAME`**，可选(建议)，同上
- `CF_PAGE_BRANCH`，可选，空或 `main` 值对应 production 环境，否则为 preview 环境
- `CF_NAMESPACE_ID`，可选，若在 Secrets 中未设置，则必须在此处设置

以部署 page 举例，除必填项 **`CF_ACCOUNT_ID` `CF_API_TOKEN` `CF_NAMESPACE_ID`**  外，

- 如果设置的 **`CF_PAGE_NAME`** 未在 cloudflare dash 创建过，对于首次部署建议设置好  **`CF_PAGE_UUID`** ；若已创建过且cloudflare dash 后台 page 页 settings 已设置 UUID 环境变量的话，这里可忽略 `CF_PAGE_UUID`

- 设置好 Secrets 和 Variables 后，左侧 'Actions' 下拉点击 'General'，右边下方"Workflow permissions"下勾选"Read and write permissions"，然后点击"save"保存；

- 之后导航栏点击 "Actions" 页，点击 enable(启用) 按钮，左侧 *cfhost Workflow* 右侧 *enable workflow* （*Upstream Sync* 同理也需要启用！）

- 手动触发一次部署：Actions - cfhost workflow 页面右侧点"Run workflow"，等待出现✅（也可点击执行记录进入查看执行结果）之后到cloudflare访问对应worker/page域名，如果正常访问则说明已成功！

  ~~如果你急需要筛选过的有效反代IP 数据，可到本仓库 https://github.com/geeksnail/edgetunnel/actions/workflows/update_ip.yml 点击第一条运行记录，左侧 *Jobs* - *deploy* - *build-deploy*，右侧 *prepare proxys data*，复制单引号内 {xxx} 数据；回到你仓库 Actions 页面，左侧 cfhost workflow，右侧 Run workflow，粘贴到 'proxys json data' 框中，手动触发运行~~

两个工作流(Workflow)说明：

- ~~IP Workflow~~：每周触发2次 下载上游IP更新数据，每天触发1次 合并反向代理 IP 并测试，将有效的反代IP数据更新到 src/proxys.json ，并调用 cfhost 工作流继续执行；(仅由原仓库触发执行 避免重复浪费资源)

- cfhost Workflow：每2小时触发一次，测试并过滤 KV-cfhost 数据；当由上述 IP 工作流每天触发调用时，将 KV-cfhost 数据合并到 `cfhost.json`，连同 `proxys.json` 集成到 _worker.js 完成构建-部署。

- Upstream Sync：每天触发1次，与上游即本仓库代码保持同步，包括每日可能更新的proxys.json数据与本人不定期的代码优化更新，若有变动则触发执行 cfhost 工作流。

  注：当上游主仓库".git/workflows"目录下的workflow文件`.yml`更新时，由于 GitHub 限制，Upstream Sync工作流无法自动同步 workflow 文件，Actions页显示Sync失败❌！你可选择以下任一操作来解决：

  1. (手动) 当每次 Upstream Sync 执行失败时，回到仓库页，点击 **Sync fork → Update branch** 来手动更新。

  2. (自动) 在 workflow 执行中更新 workflow 文件需要创建一个"Personal Access Token"，https://github.com/settings/tokens/new，（如图）Note注释随意填，Expiration(到期)选择"Custom"或"No Expiration"，Scopes勾选"repo"和"workflow"两个范围权限即可，其中workflow权限就是为了解决上述问题（可用于在执行workflow时更新workflow文件）；之后点击"Generate token"，会生成一串`ghp_xxxxxx`格式的token，记得复制备份（只显示一次）！

     回到仓库 → **Settings** → **Secrets and variables** → **Actions**，点击 **New repository secret**，

     名称 **`GH_SYNC_TOKEN`**，secret值：粘贴刚才生成的 token。可手动运行一次Upstream Sync验证

     ![image-20250922040251710](https://s2.loli.net/2025/09/22/BSp1tAaVHjGnFL9.png)

### 部署多个worker/page

设置仓库变量(Repository Secrets) `CF_WORKER_NAME` 或 `CF_PAGE_NAME` 时，以空格或换行分隔

设置仓库密钥(Repository Variables) `CF_WORKER_UUID` 或 `CF_PAGE_UUID` 时，以空格或换行分隔

注：以 page 为例，若 `CF_PAGE_UUID` 里 UUID 数量少于 `CF_PAGE_NAME` 里 page name 数量，则 `CF_PAGE_NAME` 里剩余没有 UUID 输入对应的 page name 部署时 将采用其之前部署设置过的已存在的 UUID 值（即保持不变）；如果你希望在 cloudflare dash 后台设置页手动更新某个 page UUID，则需将此 page name 置于 `CF_PAGE_NAME` 变量的结尾，并在 `CF_PAGE_UUID` 值的对应尾行留空

```
CF_PAGE_NAME:
page-name1 (与 uuid1... 对应)
page-name2 (未设置对应uuid值时将使用已有值或默认值)

CF_PAGE_UUID: 
uuid1...
```

## 使用

cloudflare dash 后台到 Workers & Pages 页找到刚部署好的 worker/page 实例， 可以看到完整域名

worker 域名格式：`<worker_name>.<subdomain>.workers.dev`

page 域名格式：`<page_name>.pages.dev` or `<page_name>-xxx.pages.dev`

几个链接：

- 伪装主页：https://<worker_or_page_domain>/

- 测试页：`https://<worker_or_page_domain>/cf` or `https://<worker_or_page_domain>/cdn-cgi/trace`

  check key value such as 'country','colo',timezone'

- 项目主页：`https://<worker_or_page_domain>/<UUID>`

- **vless默认订阅**：`https://<worker_or_page_domain>/sub/<UUID>` （地址含优选域名和IP）

- vless优选IP订阅：`https://<worker_or_page_domain>/bestip/<UUID>` （遗留，用的cmliu的接口，延迟高，不建议使用！）

如果你的网络不支持访问 **worker** 域名, 尝试用托管在 cloudflare 的特定域名(如`ip.sb`) 来构造curl命令：

```sh
# get worker vless default config:
curl -s -H 'host:<your_worker_domain>' http://ip.sb/<UUID>
# response example
# vless://<UUID>@<address>:443?security=none&encryption=none&fp=randomized&type=ws&host=<host>&path=%2F%3Fed%3D2048#<remark>

# get worker vless subscription config list:
curl -s -H 'host:<your_worker_domain>' http://ip.sb/sub/<UUID> > sub.txt
```

### 客户端设置

客户端参数设置里切记要**关闭**"默认跳过证书验证(**allowInsecure**)"，然后再更新当前分组订阅！

下面介绍的参数是为方便理解，实际无需手动配置！

- 地址(address) ：个人推荐 优选域名 或 优选 cf ip，默认自带的 page 域名适合备用
  - worker 域名或 page 域名，如果你的网络支持访问。（page 域名经测试只有移动网络访问有时不稳定）
  - 自定义域名：worker 自定义域名得绑定你在 cloudflare 里的已有域名，绑定后即支持 https；page 域名自带 https，自定义域名更方便，如使用 https://dynv6.com/ 的免费域名
    ```sh
    # 如对域名 test.dns.army 添加一条类型 CNAME 的别名记录 t，使域名 t.test.dns.army 解析为 ttt-8a4.pages.dev
    Name:t 
    Data:ttt-8a4.pages.dev.
    ```
  - 优选域名：代码默认用由 cf 托管的知名网站域名(生成订阅)，可自行到支持批量 ping 测试的网站优选域名，优点是保证至少若干个域名能稳定连通， 缺点是多了解析 ip 的过程 连接延迟不保证长期稳定
  - 优选 cf cdn/anycast ip: https://github.com/XIU2/CloudflareSpeedTest
  
  ```sh
  # 延迟测试命令举例: 平均延迟上限100ms 丢包率上限20% 禁用下载测速
  ./CloudflareST -tl 100 -tlr 0.2 -dd 
  ```
  
- 端口(port)：7个http端口可任意选择(80、8080、8880、2052、2082、2086、2095)，6个https端口可任意选择(443、8443、2053、2083、2087、2096)；注：page必须用https端口

- 用户ID(uuid)：上面提到的 仓库 Settings - Secrets 里填的 `CF_WORKER_UUID` 或 `CF_PAGE_UUID`，或在 Cloudflare dash worker/page 页面设置的环境变量 `UUID` 值

- 传输协议(network)：选 ws 或者 websocket

- 伪装域名(host)：worker 域名或 page 域名或自定义域名

- 路径(path)：/

- 传输安全(TLS)：端口填http的，选不开启或空；端口填https的，选开启 tls

- SNI：与伪装域名(host)一致

~~注：如果伪装域名填自带的 worker 域名，且开启了 TLS，则需要客户端设置中启用分片(Fragment)功能，其中 xray 的两个客户端版本如下，其他客户端自行查找~~

~~v2rayN >=6.43 https://github.com/2dust/v2rayN/releases~~

~~v2rayNG >= 1.8.17 https://github.com/2dust/v2rayNG/releases~~

## 问题汇总

- **首次执行cfhost workflow并未在cloudflare看到新部署的worker/page，deploy 阶段输出错误：**

  ```
  create_page success (或 create_worker success)
  .....
  { success: false, errors: [{ code: 8000007, message: Project not found. }] }
  ```

  大概率是你在 `CF_WORKER_NAME` / `CF_PAGE_NAME` 中起的名已被占用了，命令行检查完整域名：

  ```sh
  # 若是 deploy worker 报错
  nslookup <worker_name>.<subdomain>.workers.dev
  # 若是 deploy page 报错
  nslookup <page_name>.pages.dev
  ```

  若输出结果包含正常IP，说明名字已被占用！请重新取名后再运行一次 cfhost workflow！

- **客户端获取 page 订阅时提示失败**

  目前仅在部分移动网络区域遇到。可在浏览器访问订阅链接，手动拷贝内容粘贴到客户端；也可通过配置使用自定义域名解决

- **v2ray 客户端真链接全选测试延迟 -1，或选几个测试一下正常 一下又显示 -1**

  先对 cloudflare 上的一些网站域名的 443 端口测试：

  ```sh
  telnet ip.sb 443
  # or
  curl https://ip.sb
  ```

  若 telnet 不能通或偶尔能通，或者 curl 测试卡顿或偶尔能返回 ipv4 地址，说明你的网络通过 ipv4 地址访问 cloudflare 网段不稳定！

  如果你的光猫没有公网ipv4地址，而有ipv6地址(24开头)，但是机器却没有分配到公网 ipv6 地址，请检查路由器是否开启 ipv6！（ipv6 连通性相对更好）

  注：PC端尤其是 worker 每次测试不要全选，虽然启用分片后 worker tls 链接可以用，但是其真链接测试 右侧仍显示-1，对 worker 链接的批量测试一般只选中非 tls 部分！！！

- **访问某些由 cloudflare 托管的网站如 chatgpt.com 加载失败**

  某些网站支持 http3/quic 协议，导致浏览器会发 UDP 请求，而 cloudflare worker 暂不支持 UDP。

  https://en.wikipedia.org/wiki/HTTP/3

  ![image-20240716014603558](https://s2.loli.net/2024/07/16/vyXletgExaNHqQP.png)

  解决：若该网站必须通过代理才能访问，则关闭 quic：chrome 浏览器复制 chrome://flags 到地址栏回车，搜索 quic，禁用 quic 协议

  ![image-20240716014250950](https://s2.loli.net/2024/07/16/c9k1TwavblefDdC.png)

- **访问某些由 cloudflare 官方或由其托管的网站报错 `403 forbidden cloudflare`**

  虽然上述 IP 工作流 每天都会定时测试反代IP有效性，但是在日常使用中，某些反代服务器转发请求时仍有一定概率违反了 cloudflare 相关规则，如 SSL 证书错误。尝试刷新几次，若问题依旧，则说明浏览器与目标网站一直复用错误连接，导致刷新也没用。

  解决：chrome浏览器复制 [chrome://net-internals/#sockets](chrome://net-internals/#sockets) 到地址栏回车，点击 'close idle sockets'，可选点击 'Flush socket pools'
  
- ~~**npm install 报错 403 forbidden。（registry.npmjs.com 由 cloudflare 托管代理）**~~

  ~~解决：建议使用 pnpm，重试更有效率~~ （已将 registry.npmjs.com 放入 cfhost.json，故解决）
  
- **Upstream Sync工作流执行失败，与主仓库同步更新失败**

  参照下图手动同步更新，点击"**Sync fork**" 再点"**Discard**"按钮
  
- **"1101"异常错误，所有服务均无法访问！**

  可能是对应的 worker/page 运行的**代码**在部署时被 cloudflare 检测到敏感词/串导致的封禁

  解决：首先 1. cloudflare删除被封禁的worker与page，
  
  若要保持原有仓库及action变量设置：
  
    2 若被封禁的是page，则到github对应仓库Settings下(如上面图1所示) Variables页，更改`CF_PAGE_NAME`变量值（原来的page名称不能再用！worker名不影响）
  
    3 github仓库页如下图点击"**Sync fork**" 再点"**Update branch**"或"**Discard**"按钮... 与本仓库最新代码保持同步（随后将自动触发 cfhost workflow，等待完成部署即可）
  
  若要用新仓库重新配置：
  
    2 github上fork本仓库，Settings - Secrets and Variables下重新设置好上面提到(或参考之前设置过)的必要参数变量
  
    3 github仓库action页手动触发执行一次 cfhost workflow 完成部署
  
  建议：至少部署一个worker，确认正常访问后将worker里的代码拷贝，手动再部署一个page或worker作为备用（手动部署因为仅部署一次 所以可完全规避1101问题，不过集成的数据随着时间可能存在滞后！）
  
  ![image-20250919190009169](https://s2.loli.net/2025/09/19/8yqKjuVAvoHsUgP.png)
  
  ![image-20250902204806533](https://s2.loli.net/2025/09/02/2lFrX9TZvhtPnx8.png)

## 简图

```
              ┌-------------------------┐
ISP ⇠cf cdn⇢ |        worker           | ⇠(direct)⇢ remote
--------------┼-------------------------┼-------------------------------
              | (direct) incoming data ?├┈┈no┈→ proxy ←┈cf cdn┈→ remote
              └-------------------------┘
after:

              ┌--------------------------┐  no ┈┈(direct)┈→ remote
ISP ⇠cf cdn⇢ |        worker            |  /                   ↕
              | remote in cfcidr/cfhost ?├ ┈┈yes┈→ Hit proxy ←→ cf cdn
--------------┴-----------------------⇡--┴-------------------------------
               tag remote into cfhost ┊no
              ┌-----------------------┴--┐
              | (direct) incoming data ? ├┈┈no┈→ proxy ←┈cf cdn┈→ remote
              └--------------------------┘
```

## Reference Links

- https://github.com/zizifn/edgetunnel
- https://github.com/6Kmfi6HP/EDtunnel
- https://github.com/yonggekkk/Cloudflare-workers-pages-vless
- https://blog.cmliussss.com/p/CloudFlare%E4%BC%98%E9%80%89/
- https://subdomainfinder.c99.nl/ Subdomain Finder and CloudFlare Resolver
- https://myip.ms/browse/sites/1/ownerID/4638/ownerID_A/1  World IP Address Owners: Cloudflare Inc

## 捐赠/赞赏

<span style="display:flex"><img src="../../images/wechat_reward_qrcode.png" style="width:51%"/><img src="../../images/alipay_qrcode.jpg" style="width:49%"/></span>

## 联系

Email：geeksnail@outlook.com

QQ：353988079

闲鱼/小红书：且听风吟（头像对应本站 logo）

本仓库 [GitHub Discussions 留言板](https://github.com/GeekSnail/edgetunnel/discussions) (同下方)

## 留言板
<div class="giscus"></div>
<script src="https://giscus.app/client.js"
        data-repo="GeekSnail/edgetunnel"
        data-repo-id="R_kgDOMCvhdA"
        data-category="General"
        data-category-id="DIC_kwDOMCvhdM4Cgs8H"
        data-mapping="title"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="1"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
		data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>
