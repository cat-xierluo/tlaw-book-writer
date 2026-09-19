# tlaw-book-writer

书稿写手技能——把要写的章节交给 AI agent，它按你配置的范稿风格与写作规范直接成稿，交稿前自动跑规范自查。适用于个人书稿或多人合著项目（支持飞书文档协作场景）。

## 使用

1. Clone 本仓（或下载 ZIP）。
2. 首次对 agent 说"写我的 X.X"或"我的下一节"——会引导你完成个人配置（署名 + 在写的章节；连接了飞书 CLI 的可从多维表格书稿计划读取并与你确认），自动写入 `config/project-profile.yaml`（已 .gitignore）。
3. 按 `config/project-profile.example.yaml` 的注释配置配套材料（写作规范 / 范稿 / 词汇清单）后，技能以完整模式运行；未配置时降级运行（跳过风格基线与更新哨兵）。

飞书 CLI 为可选增强（读取知识库 / 规范更新哨兵）：见 `references/feishu-cli-cookbook.md` §0，约 5 分钟上手。

## 贡献（PR 欢迎）

- 欢迎对**技能流程与检查方法**提 Issue / PR：写作流程（备料三读 / 事实边界 / 自查）、检查表结构、飞书 CLI 手册命令。
- **不要**在 PR 中提交：真实飞书 token、本地配置（project-profile.yaml）、受版权保护的规范或范稿全文。

## License

MIT
