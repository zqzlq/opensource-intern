# 俱乐部开源实习课题任务书

## 一、课题基本信息

- **课题名称**：基于 PoW 的镜像站防盗刷流量机制
- **指导导师**：慕冬亮，刘靖南
- **开发语言**：TypeScript、Golang
- **预计项目时长**：2 个月
- **难度等级**：中等

## 二、技能要求

- 了解 Git 版本控制与 Github 开源协作流程
- 掌握 TypeScript 和 Golang 编程
- 对 Nginx 工作原理有一定了解
- 了解常用哈希函数与 PoW 机制

## 三、课堂背景

在国内 PCDN 盛行的情况下，一些不法人员为了平衡上传/下载流量，避免被运营商限速或关停，会大量请求大型开源镜像站中分发的发行版安装ISO镜像以增加下行流量。这不仅给镜像站带来了极大的服务负载，同时也挤占了其他合法用户的带宽。

为了解决以上问题，不同镜像站采取了以下两种主流的解决方案：

- 以 TUNA 为例的镜像站采用了流量监控+防火墙策略，若检测到来自某IP段的流量在一段时间内超过阈值，则将其纳入防火墙封禁。这样做的优点是对正常用户几乎无感且不要求镜像站暴露额外API，缺点是部分用户可能因为网段内存在恶意行为而被一并封禁，且无法及时被解封。
- 以 USTC 为例的镜像站使用了请求风险检测技术，如 Anubis。在用户请求资源前，服务器会要求用户在本地进行一些工作量证明或完成人机验证，否则将阻断请求。这样做的优点是能够细粒度的防护且极大的降低了误封的风险，缺点是一旦使用需要前端浏览器交互进行的验证，wget curl等请求工具将无法正常下载，对无头环境的用户造成了很大困扰，且需要涉及动态页面和后端的暴露，存在网络安全隐患。

与其他网站不同，开源镜像站具有一定的特殊属性：我们既需要保证开放、低门槛、兼容 wget/curl/包管理器等工具，又要防止大文件被滥用。现有的两种方案都无法完全满足镜像站的需求。

## 四、课题任务

为了在尽可能保证用户体验的情况下防范攻击，我们提议开发一种实现人机验证和流量验证分离的防护手段。

我们可选择常规人机验证手段（如验证码、极验），也可以选择基于工作量的证明（PoW）。由于大多数学校网络中心对于可对外访问的系统有较为严格的限制，使用传统验证手段所需要的后端 API 服务可能无法放行，故本课题的目标为开发一套基于工作量证明的验证手段。

- 任务 1：前端生成签名
  - 为镜像站前端增加 在下载大文件（例如特定后缀）时，自动弹窗提示用户进行工作量验证
  - 生成结束后提示用户如何使用带有 signature 的下载 url
- 任务 2：后端 signature 验证
  - 验证请求是否需要签名验证，如果是的话请求中是否存在 signature
  - 验证 signature 是否符合规定工作量，是否合法，是否已超过使用次数
  - 将使用过的 signature 写入数据库
- 任务 3：Nginx 服务配置
  - 搭建镜像站影子站点以供测试
  - 修改 Nginx 配置文件以接入验证后端
- 任务 4：测试
  - 测试 非法请求能否被正确拦截
  - 测试 经过签名后的 url 是否能正常在无头环境下使用
  - 测试 选择的签名算法与难度是否合适

<details>
  <summary>以下为供参考的任务流程图</summary>
  
  ```mermaid
  graph TD
      Browser["🖥️ 用户浏览器
  前端 PoW 验证模块"]
      Nginx["⚙️ Nginx 反向代理
  auth_request 鉴权模块"]
      Backend["🐍 PoW 验证后端
  Python 签名校验服务"]
      Redis[("💾 Redis / 内存
  已用 Sign 记录")]
      FileServer["📦 文件服务
  ISO / 大文件存储"]
  
      Browser -->|"HTTPS 请求
  携带 token + sign"| Nginx
      Nginx -->|"内部子请求
  X-Original-URI, X-Real-IP"| Backend
      Backend -->|"读写 Sign 使用记录
  （通用链接模式）"| Redis
      Backend -->|"HTTP 200 放行
  / HTTP 403 拒绝"| Nginx
      Nginx -->|"200 → 转发请求"| FileServer
      FileServer -->|"文件数据流"| Browser
  ```

  ```mermaid
  sequenceDiagram
      actor User as 👤 用户
      participant FE as 前端 PoW 模块
      participant Nginx as Nginx
      participant Auth as PoW 验证后端
      participant Redis as Redis
      participant FS as 文件服务
  
      User->>FE: 点击下载大文件（如 .iso）
      FE-->>User: 弹出验证窗口，选择模式\nA: 绑定IP（长效）/ B: 通用链接（短效）
      User->>FE: 确认 IP 与目标路径
  
      Note over FE: 构造签名字符串\n"{ip}|{path}|{timestamp}|"
  
      loop PoW 计算（Web Crypto API）
          FE->>FE: SHA-256("{签名字符串}{cnt}")\n检查哈希前 N 位是否全为 0
      end
  
      FE-->>User: 生成下载 URL\n?token={base64}&sign={hash}\n展示一键复制 / 立即访问
  
      User->>Nginx: GET /ubuntu.iso?token=...&sign=...
  
      Nginx->>Auth: 内部子请求 /verify_pow\n（X-Original-URI, X-Real-IP）
  
      Auth->>Auth: ① token / sign 参数是否存在？
      Auth->>Auth: ② sign 是否满足 PoW 难度？
      Auth->>Auth: ③ token 中路径与请求路径是否匹配？
      Auth->>Auth: ④ 时间戳是否在有效期内？
  
      alt 模式 B：通用链接
          Auth->>Redis: 查询 sign 使用次数
          Redis-->>Auth: 返回计数
          Auth->>Auth: ⑤ 使用次数 < N？
          Auth->>Redis: 写入 / 递增计数
      else 模式 A：绑定 IP
          Auth->>Auth: ⑤ 请求 IP 与 token 中 IP 是否一致？
      end
  
      alt 全部校验通过
          Auth-->>Nginx: HTTP 200 OK
          Nginx->>FS: 转发下载请求
          FS-->>User: 文件数据流 ✅
      else 任意校验失败
          Auth-->>Nginx: HTTP 403 Forbidden
          Nginx-->>User: 拒绝并返回验证引导页 ❌
      end
  ```
</details>

## 五、课题验收

本项目预计于认领后1-2个月内开展验收，验收项如下所示：

- **代码**：完整的功能实现，提交至 Github 相关仓库并被合并
- **文档**：部署手册、维护文档
- **汇报**：课题总结 PPT 或技术博客。鼓励以博客方式发布，便于社区传播与知识沉淀
- **测试**：在本地环境通过功能测试；能够正确拦截未通过验证的请求；在 curl wget 等命令行工具中能够正常下载
