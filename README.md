# all-skills

DeepSeek Harness（DSH）技能集合。每个子目录是一个可直接安装的 skill，包含 `SKILL.md` 与按需加载的 `references/`、`templates/`。

## 技能列表

| 技能 | 用途 |
|---|---|
| [`kernel-dev`](kernel-dev/SKILL.md) | 内核 / 固件 C 工程审阅与实现助手。按 P0–P3 分级审阅，内置内存与位域判据（bit vs byte、packed、指针 vs 拷贝、一行式），覆盖 Linux 内核与 BROM/BL2/BL31/U-Boot 裸机双模式的上下文规则。 |

## 安装

复制子目录到 DSH 的用户级技能目录：

```powershell
Copy-Item .\kernel-dev "$env:USERPROFILE\.dsh\skills\kernel-dev" -Recurse
```

技能目录由 DSH 的文件监控自动发现，无需重启；下次会话即可在技能目录里看到，直接用 `/kernel-dev` 调用，或描述一个内核代码任务让它自动命中。

项目级安装则放到 `<项目根>/.dsh/skills/`（优先级高于用户级）。

## 目录约定

- 一个技能 = 一个顶层目录（或单个 `<name>.md` 文件），`name` 必须 kebab-case，且与目录名一致。
- `SKILL.md` 以 YAML frontmatter 开头：`name`、`description` 必填，可选 `whenToUse`、`disable-model-invocation`、`user-invocable`。
- `description` 是模型路由的唯一依据，控制在 500 字符以内；正文由模型按需加载，不进入常驻上下文。
- 资源文件放在技能目录内的子目录（`references/`、`templates/`、`scripts/`），不要嵌套新的 `SKILL.md`——DSH 只扫描一层。
