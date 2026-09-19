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

## §3 文档写入（⚠️ 未真机验证——首用先建测试文档）

### 路线 A（推荐待验证）：md 导入任务 → 挂载知识库

飞书导入任务原生支持 markdown → docx，块结构由平台生成（比手搓块稳）：

```bash
# 1. 上传 md 到云空间（import 专用父类型）
lark-cli api POST /open-apis/drive/v1/files/upload_all \
  --data '{"file_name":"3.4-测试.md","parent_type":"ccm_import_open","parent_node":"ccm_import_open","size":<字节数>}' \
  # ↑ 二进制上传需 -o/--file 形式，以 lark-cli drive 子命令或 schema 现查为准

# 2. 创建导入任务（md → docx；mount_type 2 = 知识库，mount_key = space_id）
lark-cli api POST /open-apis/drive/v1/import_tasks \
  --data '{"file_extension":"md","file_token":"<上一步 file_token>","point":{"mount_type":2,"mount_key":"<space_id>"},"file_name":"2.5 撰写庭审讯问提纲-张三"}'

# 3. 轮询任务结果 → 拿到 docx token
lark-cli api GET /open-apis/drive/v1/import_tasks/<ticket> --as user
```

### 路线 B（备选）：直接建节点 + 逐块写入

```bash
# 1. 在章节父节点下建 docx（parent_node_token 从章节节点链接解析）
lark-cli api POST /open-apis/wiki/v2/spaces/<space_id>/nodes \
  --data '{"obj_type":"docx","title":"2.5 撰写庭审讯问提纲-张三","parent_node_token":"<章节节点>"}'

# 2. 逐块追加正文（根块 id = document_id；block 结构先现查 schema，勿凭记忆写）
lark-cli schema docx.block.children.create          # 查块结构与 block_type 枚举
lark-cli api POST /open-apis/docx/v1/documents/<doc_id>/blocks/<doc_id>/children \
  --data '{"children":[ ... ]}'
```

**md → 块映射**（`#`→heading1、`##`→heading2、`###`→heading3、`####`→heading4、普通段→text、代码块→code、`>`→quote；具体 block_type 数值以 `lark-cli schema` 现查为准，勿硬编码）。

### 图片

```bash
# 上传原图（PNG ≥1200px；上传后再插图块；文件名与图名一致）
lark-cli api POST /open-apis/drive/v1/medias/upload_all \
  --data '{"file_name":"3.4-图02-创建技能入口.png","parent_type":"docx_image","parent_node":"<doc_id>","size":<字节数>}'
```

### 验证纪律

1. 首用任一路线：先以《X.X-通道测试-<作者名>》为题在知识库建测试文档，全流程走通（含标题样式、代码块、题注）后**删除测试文档**，方可用于正式章节。
2. 写入只用于**新建自己的章节文档**；修改一律回飞书 UI 走"修订"模式（API 修订模式不可用）。
3. 导入/建文后人工核对：H1 唯一（文章题）、编号未触发自动有序列表、代码块完整、图位与题注正确。

## §4 常用节点

真实 token 只记录在**本地实例** `config/project-profile.yaml`（.gitignore 排除——本手册不含任何真实 token）。获取方法：知识库/文档链接中 `wiki/<node_token>` 段；obj_token 用 §1 的 get_node 解析。缺节点（术语表/进度表/样文/各章父节点）时向知识库维护者获取链接。
