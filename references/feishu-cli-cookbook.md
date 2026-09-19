# 飞书 CLI 操作手册（feishu-cli-cookbook）

> 面向 tlaw-book-writer 的 lark-cli 用法：连接上手、活体读取、规范哨兵、归档、文档写入。命令按 lark-cli 1.0.41 语法写出；**各节标注验证状态**——未验证的写入链路首用前必须先建测试文档试跑。

## §0 连接上手（从零到能读，约 5 分钟；lark-cli 为可选增强——不装也能用本技能，交付走通道 a 手动粘贴）

```bash
# 1. 安装（任选其一；需 Node 18+）
npm install -g @larksuite/cli          # 官方仓库 github.com/larksuite/cli
# （macOS 亦可用 Homebrew：brew install lark-cli，若 formula 可用）

# 2. 首次授权（浏览器里用你的飞书账号确认；会绑定一个应用身份）
lark-cli auth login

# 3. 自检三连：状态 user ready → 解析知识库节点 → 读规范全文
lark-cli auth status --jq '.identities.user.status'          # 期望输出 ready
lark-cli api GET /open-apis/wiki/v2/spaces/get_node \
  --params '{"token":"<知识库 wiki token，见你收到的项目链接>"}' --as user \
  --jq '.data.node | {title, obj_type, obj_token, space_id}'
lark-cli api GET /open-apis/docx/v1/documents/<规范 obj_token>/raw_content \
  --as user --jq '.data.content'
```

**配置**：复制 `config/project-profile.example.yaml` 为 `config/project-profile.yaml`，填入带 `# 待填` 标记的字段（你的署名、在写的章节；飞书 token 按上一步解析结果填入），其余按你的书稿配置。

**常见问题**：`need_user_authorization` / user missing → 重跑 `lark-cli auth login`；`131006 wiki space permission denied`（整库枚举）→ 正常，按 token 读单篇文档不受影响；命令报更新提示 → 可跑 `lark-cli update`。

## §1 活体读取（✅ 已验证，260919）

```bash
# 状态检查（user 是否 ready）
lark-cli auth status --jq '.identities.user.status'

# wiki 链接 → 节点信息（title / obj_type / obj_token / space_id）
lark-cli api GET /open-apis/wiki/v2/spaces/get_node \
  --params '{"token":"<wiki_node_token>"}' --as user \
  --jq '.data.node | {title, obj_type, obj_token, space_id}'

# 读文档全文（纯文本，丢失样式结构）
lark-cli api GET /open-apis/docx/v1/documents/<obj_token>/raw_content \
  --as user --jq '.data.content'

# 需要结构（标题层级/表格/代码块）时读块列表并自行重建
lark-cli api GET /open-apis/docx/v1/documents/<obj_token>/blocks \
  --params '{"page_size":500}' --as user --page-all
```

已知限制：知识库整库枚举（`wiki/v2/spaces/{id}/nodes`）返回 131006（space 级权限未授予）——按 token 读文档不受影响；要枚举需在飞书管理端给该应用开 space 读权限，或由人把节点链接交给 agent。

## §1.5 读书稿章节计划（⚠️ 待验证——需表格链接；首次引导的"库读取"用）

```bash
# 进度表（多维表格）链接 → 解析（链接形如 …/wiki/<node> 或 …/base/<app_token>）
lark-cli api GET /open-apis/wiki/v2/spaces/get_node \
  --params '{"token":"<进度表节点 token，见 profile.feishu.progress_token>"}' --as user \
  --jq '.data.node | {obj_type, obj_token}'

# obj_type=bitable → 列出数据表，取记录（"作者"字段 × 章节字段）
lark-cli api GET /open-apis/bitable/v1/apps/<app_token>/tables --as user --jq '.data.items[].table_id'
lark-cli api GET /open-apis/bitable/v1/apps/<app_token>/tables/<table_id>/records \
  --params '{"page_size":100}' --as user --page-all --jq '.data.items[] | .fields'

# 备选：从章节父节点下文档标题的"-作者"后缀推断作者与章节的对应（需 chapter_nodes 已配；space 级枚举受限时不可用）
lark-cli api GET /open-apis/wiki/v2/spaces/<space_id>/nodes \
  --params '{"parent_node_token":"<章节节点>"}' --as user --jq '.data.items[].title'
```

读到的结果**必须与用户二次确认**后再写入 author 块（同名/代填风险）。

## §2 规范哨兵（✅ 已验证）

```bash
# 规范 wiki 的 revision_id（本书基线见 profile.feishu.revision_baseline）
lark-cli api GET /open-apis/docx/v1/documents/<standards_obj_token> \
  --as user --jq '.data.document.revision_id'
```

涨过基线 = 规范被静默修改 → 重拉 raw_content、diff 本地 V1.1 归档、更新 checklist/profile、抬高基线，然后才继续写作（SKILL §2bis）。


## §3 文档写入（⚠️ wiki 写入受组织限制——详见 §3.0；云盘路径 ✅ 已验证）

### §3.0 已知限制：wiki 空间写入的组织门槛

**问题**：飞书 wiki 空间的建节点/移入文档等写入操作，要求 API 调用者必须是**空间所在组织的成员**。如果你的飞书账号不在该组织——即使有正确的 OAuth scope、即使浏览器里能正常编辑文档——API 写入仍会报 `131006 permission denied`。

**根因**：飞书 OAuth token 按**组织**（tenant）签发；跨组织访问走的是浏览器的跨租户分享通道，API 走的是组织内的 token 校验，两条通道的权限模型不同。

**自查方法**：

```bash
# 查看你的 tenant_key
lark-cli api GET /open-apis/authen/v1/user_info --as user --jq '.data.tenant_key'
# 对比知识库 URL 的域名前缀（如 xxx.feishu.cn → 组织前缀 xxx）
# 不一致 = 跨组织，wiki 写入会被拦
```

**解决方式**：让知识库管理员把你拉进空间所在的飞书组织，然后 `lark-cli auth login --domain wiki` 重新授权。

### §3.1 推荐路径（✅ 已验证）：云盘建 docx + 写入内容

**绕过 wiki 限制**——直接在项目的云盘文件夹（drive）里建飞书文档，写入内容：

```bash
# 1. 在项目云盘指定文件夹里建 docx（folder_token 从 profile.feishu 获取）
lark-cli api POST /open-apis/docx/v1/documents \
  --data '{"title":"<文档标题>","folder_token":"<项目云盘文件夹 token>"}'

# 1.5 写入前清理：去除 md 标记；表格转文字要点；无截图的图位删除

# 2. 逐块写入内容（root block id = document_id）
#    md → 块映射：#→heading1(block_type:3) / ##→heading2(4) / ###→heading3(5) / ####→heading4(6)
#    普通段→text(2)、代码块→code(14)、引用→quote、图片→media(27)
lark-cli api POST /open-apis/docx/v1/documents/<doc_id>/blocks/<doc_id>/children \
  --data '{"children":[ ...blocks... ]}'

# 3. 读回验证
lark-cli api GET /open-apis/docx/v1/documents/<doc_id>/raw_content --as user
```

**已验证结论**：
- ✅ 建云文档（个人空间/项目云盘根级文件夹）
- ✅ 写入 heading1-4 / text / code / quote 块
- ✅ 读回内容完整
- ❌ 移入 wiki 空间/章节子文件夹（组织限制）
- 文档出现在云盘指定位置，浏览器可见；如需放入 wiki 章节，手动拖入

### §3.2 备选（⚠️ 受 §3.0 组织限制）：md 导入任务 → 挂载知识库

飞书导入任务原生支持 markdown → docx，块结构由平台生成。**前提：你的账号在 wiki 空间所在组织内。**

## §4 常用节点

真实 token 只记录在**本地实例** `config/project-profile.yaml`（.gitignore 排除——本手册不含任何真实 token）。获取方法：知识库/文档链接中 `wiki/<node_token>` 段；obj_token 用 §1 的 get_node 解析。缺节点（术语表/进度表/样文/各章父节点）时向知识库维护者获取链接。
