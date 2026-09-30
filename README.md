# 第 4 课 OpenSpec 变更

本仓库只包含第四次课的变更描述，不包含业务实现代码。

变更目录：

```text
openspec/changes/add-traceable-vector-retrieval/
├── proposal.md
├── design.md
├── tasks.md
└── specs/knowledge-retrieval/spec.md
```

能力名称是 `knowledge-retrieval`。它在第 3 课已有的登录、班级隔离和材料入库之上，增加限定本班的知识库检索，并让每条命中回溯到材料、切片和字符偏移。切片正文放在 MySQL，向量放在向量数据库。
