---
name: feishu-docs
description: "读飞书 / Lark 的文档、知识库、表格、多维表格。触发：贴来 feishu.cn / larksuite.com / larkoffice.com 链接、读一下这个飞书文档、把这个表格拉出来。"
---

# 读飞书文档

## 用你自己的飞书身份读
以「用户身份」调接口，能读到你本人在飞书里能看到的一切（包括别的公司分享给你的文档）。

1. 装官方命令行：`npm i -g @larksuite/cli`（没有 Node 就先装 Node），然后按提示登录一次你的飞书账号。
2. 判断链接类型（去掉 `?` 和 `#` 后面的部分）：
   - `/wiki/<token>` → 先 `GET /open-apis/wiki/v2/spaces/get_node?token=<token>` 换成真实文档 token 和类型；
   - `/docx/<token>` → `GET /open-apis/docx/v1/documents/<token>/raw_content` 拿正文；
   - `/sheets/<token>` → 先列出每个 sheet，再按范围读前几十行；
   - `/base/<token>` → 多维表格，列出数据表，再读记录。
3. 命令形如 `lark-cli api GET <路径> --as user`。

## 读不到时
- 报 permission / not found：这份文档没分享给你。如实告诉用户，请对方「分享给我」或打开「链接可阅读」，不要假装读到了。
- 子文档里的链接 raw_content 拿不到：改用 blocks 接口（`/documents/<token>/blocks?page_size=500`）把链接捞出来。

## 交付
先给要点摘要，再按需要给全文或表格；表格转成 CSV/Excel 存到当前目录。
