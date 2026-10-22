# 探查方法

与语言无关。先认生态，再用该生态的常规地方下手；不确定是什么语言时，列目录就能看出来。

**命令要合并且带上结论。** 下面每一步都可以把多个探测并成一条命令；不要为"有没有 X"逐个关键词试。判断一条命令该不该跑：**它能不能改变文档里的某一行**，不能就不跑。

## 第一步：认形

```bash
ls -la                                              # 根目录
ls -d .git .svn .hg 2>/dev/null                     # 版本控制类型
find . -maxdepth 3 \( -name "README*" -o -name "CONTRIBUTING*" \
  -o -name "CLAUDE.md" -o -name "AGENTS.md" \) -not -path "*/node_modules/*" | head -30
find . -maxdepth 3 -type d \( -name doc -o -name docs -o -name design -o -name deploy \) 2>/dev/null
```

忽略：`node_modules`、`vendor`、`target`、`dist`、`build`、`.gradle`、`venv`、`.venv`、`__pycache__`、`.next`。

先把根目录、版本控制、构建清单、文档目录**一次看全**，再决定要不要派子代理。

## 第二步：按生态找构建清单

构建清单在哪，模块边界就在哪。

| 生态 | 清单文件 | 运行时版本看 | 依赖版本看 |
|------|----------|-------------|-----------|
| Node | `package.json` | `engines`、`.nvmrc`、`.node-version` | 锁定文件（`*-lock.json` / `pnpm-lock.yaml` / `yarn.lock`） |
| Java/Kotlin | `pom.xml`、`build.gradle(.kts)`、`settings.gradle` | `java.version`、`sourceCompatibility`、`kotlin.jvmTarget` | 父 POM、版本属性、BOM |
| Python | `pyproject.toml`、`requirements*.txt`、`Pipfile`、`poetry.lock` | `requires-python`、`.python-version` | `poetry.lock` / `uv.lock` / 固定版本行 |
| Go | `go.mod`、`go.sum` | `go` 指令 | `go.sum` |
| Rust | `Cargo.toml`、`Cargo.lock` | `rust-version`、`rust-toolchain*` | `Cargo.lock` |
| .NET | `*.sln`、`*.csproj`、`Directory.Build.props` | `TargetFramework` | `PackageReference` + 中央包管理文件 |
| PHP | `composer.json` | `require.php` | `composer.lock` |
| Ruby | `Gemfile` | `.ruby-version` | `Gemfile.lock` |

**版本号只从这些地方取。** 记不清某个框架当前版本是几，就去文件里查，不要写。

## 第三步：找约定

不区分语言，按"东西放在哪"搜：

| 要找 | 通常在哪 |
|------|----------|
| 对外接口 | 文件名 / 目录含 controller、handler、router、routes、api、endpoint、view |
| 业务逻辑 | service、usecase、domain、core、internal |
| 数据访问 | repository、dao、mapper、model、store、db、schema |
| 数据结构定义 | entity、models、types、schema、dto、vo、contract |
| 全局错误处理 | middleware、interceptor、filter、error、exception |
| 鉴权 | auth、security、permission、guard、middleware |
| 配置 | config、settings、env、application*、*.yaml |
| 公共工具 | utils、common、shared、lib、helpers |
| 测试 | test、tests、spec、__tests__、e2e、integration |

**抽样规则**：同类文件取 3~5 个**不同人写的**比对，用多数派写法作为规范；少数派在正文注明"存量例外"。只读一个文件就定规范，等于把个人习惯当团队约定。

**但这些整类不读**（用 1 个样本 + grep 归纳即可，逐个读是纯开销）：

- 代码生成器的产物：`*Example`、`*Mapper`、实体 getter/setter。读 1 个 DAO 实现 + 1 份 XML 就能反推全部。
- 结构同构的文件：多个服务的同款启动脚本、多个前端的同名工具模块（`request.js`、`table.js`）——读 1 份。
- 上游项目的说明文档：`README*`、`CHANGELOG*`、`LICENSE*`——版本以构建文件为准。
- 纯转发 / 空实现的模板代码——由基类与样本覆盖。

判断标准：**这条文件的内容能不能改变文档里的某一行**。不能，就跳过。

## 第四步：找不显眼但关键的东西

- **初始化 / 迁移脚本**：`*.sql`、`migrations/`、`db/`、`seed*`、`*init*data*`、`createTable*`。记录目录按什么维度分、改配置要同步哪个文件。
- **CI 配置**：`.github/workflows/`、`Jenkinsfile`、`.gitlab-ci.yml`、`.circleci/`。CI 里的命令比 README 可信。
- **容器 / 部署**：`Dockerfile*`、`compose*.yml`、`k8s/`、`helm/`、`deploy/`、`bin/`。
- **构建脚本里的非常规开关**：跳过测试、固定产物名、特殊编译参数、被注释掉但有说明的步骤。这些是"不要擅自改"清单的来源。
- **多环境配置**：有几个环境文件、默认用哪个、本地配置是否被 gitignore（提交本地配置往往是禁止项）。

**构建 / 测试只跑一次**：这类命令单次就要几十秒。确认"能不能编译""测试跑不跑"时：

- 先读构建文件给结论，能读出来就不要实跑。
- 确需实跑，合并成一次干净构建 + 一次测试，**不要换参数反复试**。想验证某个开关是否生效，读有效 POM 或插件配置，而不是反复改参数跑。

## 第五步：写示例代码

全文最多一段，用在最不直观的那处。判断标准：这段代码对应的写法，能否仅凭看目录和读相邻文件猜出来。能猜出来的不写示例。

1. 从仓库取一段真实代码，只保留体现约定的部分。
2. 脱敏：业务字段换通用名，去掉公司名、机构号、地址、密钥、真实账号。
3. 保留原语言的原生写法、注解、缩进、命名风格。
4. 长度 5~10 行。

可信度来自"看起来就像这个仓库里的代码"，不是来自"写得漂亮"，也不是来自"每类约定都给一段"。

## 常见陷阱

| 陷阱 | 表现 | 处理 |
|------|------|------|
| 版本凭印象 | 框架大版本写错 | 一律查锁定文件 |
| 把传递依赖当直接依赖 | 技术栈表塞满没直接用的库 | 只看清单里的直接声明 |
| 默认 git | 仓库其实是 SVN / Mercurial | 先看 `.git` `.svn` `.hg` |
| 画通用架构图 | 分层很漂亮但目录里没有 | 目录树必须能列出来 |
| 只看一个文件 | 抽到异类写法 | 抽 3~5 个比对 |
| 忽略"测试被跳过" | 文档写了跑测试命令，实际不跑 | 查构建脚本的跳过开关 |
| 忽略初始化脚本 | 改了配置但新环境装不上 | 查 db / migrations 目录 |
| 忽略锁定文件和环境文件 | profile / 环境写错 | 查被忽略的多环境配置文件 |
| 源文件读不出来 | 加密 / 权限 / 二进制 | 记录降级读取方式，不要跳过核心文件 |
| 把理想当现状 | 写出了仓库里根本不存在的规范 | 每条结论回到证据 |
| 写成新人手册 | 有目录树、有版本表、有通用最佳实践，但契约字段名没写 | 篇幅给契约和禁止项，其余删掉 |
| 每类约定配一段示例 | 六段示例占掉一半篇幅，都在讲 AI 本来就会的写法 | 全文最多一段 |
| 反复验证同一开关 | 为"到底生不生效"换参数跑五六次构建 | 先读构建文件；只跑一次，其余靠配置推断 |
| 逐文件读完生成器产物 | 数百个 `*Example` / 实体逐个打开 | 1 个样本 + grep 确认多数派，整类跳过 |
| 广度扫描串行跑 | 一条命令等一次，几十次往返 | 合并命令；超过一个端就派子代理并行 |
