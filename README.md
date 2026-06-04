# qiq-tech2spec

将技术方案 / RFC / 设计文档转换为**可直接指导编码的实现规格书**的 agent skill。

## 定位

`qiq-tech2spec` 只负责一个环节：

```
技术方案文档 / RFC / Design Doc
        │
        ▼
实现规格书 `<方案名>-spec.md`
        +
覆盖率报告 `SPEC_COVERAGE.md`
```

它只产出规格书与覆盖率报告，不执行工程落地动作。

## 当前保留范围

- **规格书生成**：把方案翻译为字段级、接口级、错误码级、流程级、配置级的实现规格书。
- **SQL 自检**：对规格书中的 DDL / DML / 查询 SQL 执行内置 MUST / 重要 RECOMMEND 规则自查。
- **编码约束抽取**：把通用与 Go 特化约束写入规格书 §9。
- **覆盖率报告**：输出方案条目到规格书章节的映射，确保未覆盖项为 0 或显式豁免。

## 范围边界

- 只写 `<方案名>-spec.md` 与 `SPEC_COVERAGE.md`。
- 不修改业务代码、测试代码、配置代码或 SQL 文件。
- 不对业务实现给出构建、测试、上线放行结论。

## 目录结构

```
qiq-tech2spec/
├── SKILL.md                                  # skill 主入口
├── references/
│   ├── 01-spec-generation.md                 # 技术方案 → 实现规格书工作流
│   ├── 02-sql-self-check-rules.md            # SQL / DB 自检规则
│   ├── 03-coding-constraints-common.md       # 通用编码约束
│   └── 04-coding-constraints-go.md           # Go 特化约束
├── templates/
│   ├── <方案名>-spec.md                      # 实现规格书模板（9 个主体章节 + 附录）
│   └── SPEC_COVERAGE.md                      # 方案 → 规格书覆盖率报告模板
├── scripts/
│   └── build.sh                              # skill 打包脚本
├── LICENSE
└── README.md
```

## 运行时产物

最终交付产物**只有规格书一份**，与技术方案文档同目录；覆盖率报告等中间 / 过程产物统一收口到 `.qiqskills/tech2spec/<方案名>/`：

```
<技术方案文档所在目录>/
└── <方案名>-spec.md              # 最终交付：实现规格书（与方案同目录）

.qiqskills/tech2spec/<方案名>/    # 中间产物目录
└── SPEC_COVERAGE.md             # 覆盖率报告等过程产物
```

若用户只粘贴方案正文且没有源文档路径，则规格书与中间产物一并写入 `.qiqskills/tech2spec/<方案名>/`。

## 触发词

- 技术方案生成规格书
- 方案转规格书
- design doc to spec
- tech2spec
- 实现规格书生成

## 强约束摘要

- ✅ **只生成规格书**：产物限定为 `<方案名>-spec.md` 与 `SPEC_COVERAGE.md`。
- ✅ **9 个主体章节齐全**：项目结构、数据存储、数据结构、接口契约、核心流程、异常路径、依赖关系、配置参数、编码约束均不可为空；交付自检和待确认问题放入附录。
- ✅ **SQL MUST 零违规**：规格书包含 SQL 时必须完成自检并先修正 MUST 违规。
- ✅ **覆盖率可追溯**：方案章节、数据表、Redis key、流程图、异常项、SLO / 非功能需求都要映射到规格书章节。
- ❌ **不写代码**：不创建工程实施计划、不改业务代码、不跑业务实现校验、不输出上线放行结论。

## 当前覆盖范围

- 首期重点语言：**Go**。
- 场景：互联网后台业务系统技术方案到实现规格书的转换。
- SQL / DB 规则：内置规则副本，覆盖建表、DML / 查询、索引、安全与生产底线。

## 与技术方案评审的关系

| 阶段 | Skill | 职责 |
|---|---|---|
| 方案评审 | `qiq-backend-tech-review` | 多维度评审与上线门禁建议 |
| 规格书生成 | **`qiq-tech2spec`** | 把已明确的技术方案转换为实现规格书 |

## 打包发布

把 skill 打包为可分发的 zip：

```bash
# 默认：版本号取自 git describe（回落到日期戳）
bash scripts/build.sh

# 显式指定版本号
VERSION=v0.1.0 bash scripts/build.sh

# 仅校验产物清单与内部链接，不实际打包
bash scripts/build.sh --no-zip
```

构建产物输出到 `dist/`：

```
dist/
└── qiq-tech2spec-<version>.zip
    └── qiq-tech2spec/
        ├── SKILL.md
        ├── README.md
        ├── LICENSE
        ├── references/
        └── templates/
```

构建过程会做以下校验：

- `SKILL.md` 必须存在 `name:` / `description:` frontmatter 字段。
- 所有 markdown 内部链接（指向 `.md` / `.json`）目标文件必须存在（仅 WARN，不阻塞）。
- `dist/` 建议在 `.gitignore` 中忽略。

## License

见 `LICENSE`。
