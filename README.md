# Payroll2Feishu 手工做表版

> 工资单推送飞书OA审批 — 手工做表专用 Streamlit 应用
> 
> 本项目是 **独立的全新项目**，基于生产项目 `submit_approval_feishu`（Payroll2Feishu）的飞书 API 与解析框架，并融合了 `payrolloa` 项目的**额外汇总行**（溢缴款抵扣 / 甲方转款）解析与校验逻辑，同时新增**税费**列支持，专为「手工编写工资表」场景设计。

---

## 功能

用户通过手机号登录 → 上传一个或多个 Excel 工资表 → 系统自动解析标题、合计行、额外汇总行 → 预览 13 字段详细列表 → 执行数值校验 → 一键提交飞书 OA 审批流程。

**核心特性：**
- **手工做表适配**：支持无独立单位名称行（`unit_name_row=0`），自动从标题行提取单位名
- **额外汇总行解析**：自动识别「溢缴款抵扣」「甲方转款」行，并校验 `转款合计 = 溢缴款抵扣 + 甲方转款`
- **税费列支持**：新增「税费」列，可作为独立金额字段参与校验
- **13 字段详细列表**：飞书审批表格精确映射 13 个字段
- **严格校验模式**：校验失败时禁止提交，确保数据准确

---

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/wanghannew1/Payroll2Feishu.git
cd Payroll2Feishu
```

国内镜像：
```bash
git clone https://gitee.com/wanghannew1/Payroll2Feishu.git
```

### 2. 创建虚拟环境（推荐 uv）

```bash
uv venv
source .venv/bin/activate
```

> 如果没有 uv，可用 `python -m venv .venv` 代替。

### 3. 安装依赖

```bash
uv pip install -r requirements.txt
# 国内加速
uv pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 4. 配置环境变量

复制 `.env.example` 为 `.env` 并填写：

```env
FEISHU_APP_ID=your_app_id_here
FEISHU_APP_SECRET=your_app_secret_here
FEISHU_APPROVAL_CODE=your_approval_code_here
```

> **重要**：`FEISHU_APPROVAL_CODE` 必须对应一个**新创建的飞书审批流程**（或复制现有流程后修改），因为本项目的 13 字段表格结构与生产版不同，不能直接复用生产版的审批模板。创建流程后，首次启动时程序会自动尝试通过 API 匹配控件 ID，也可在 `config.json` 中手动填写 `feishu_form.table_sub_controls` 的 `feishu_id`。

### 5. 启动服务

```bash
streamlit run demo_app.py
```

浏览器打开 `http://localhost:8501`。

---

## 13 字段详细列表

飞书审批「薪酬发放明细」表格包含以下 13 列：

| 字段名 | 说明 | 数据来源 |
|--------|------|----------|
| 报表名称 | Excel 标题 | 第 1 行 |
| 甲方转款 | 甲方转款行金额（无溢缴款时默认等于转款合计） | 额外汇总行 |
| 溢缴款抵扣 | 溢缴款抵扣行金额 | 额外汇总行 |
| 转账合计(元) | 转款合计列合计行金额 | 合计行 |
| 税费(元) | 税费列合计行金额（外包业务增值税，甲方承担按人分摊） | 合计行 |
| 扣款合计(五险一金、单位代理费) | 扣款合计列合计行金额 | 合计行 |
| 个人所得税 | 个人所得税列合计行金额 | 合计行 |
| 个人欠款 | 个人欠款列合计行金额 | 合计行 |
| 交纳现金 | 交纳现金列合计行金额 | 合计行 |
| 个人承担代理费 | 个人承担代理费列合计行金额 | 合计行 |
| 扣工会会费 | 扣工会会费列合计行金额 | 合计行 |
| 个人其他调整 | 个人其他调整列合计行金额 | 合计行 |
| 实发工资(元) | 实发工资列合计行金额 | 合计行 |

全部币种：CNY-人民币元。

---

## 解析规则说明

### 无独立单位名称行（unit_name_row=0）

本项目面向「手工做表」，通常第 1 行为标题（如 `吉林省国盛大酒店有限公司净月团餐—分公司 2026年派遣员工5月工资明细表`），**没有第 2 行的独立单位名称行**。

配置：`excel.unit_name_row=0`

程序会从标题行通过「后缀枚举 + 向左贪婪扩展 + 取最长」算法自动提取单位名：
1. 枚举 `title_unit_suffixes` 中每个后缀在标题中的所有出现位置
2. 从该位置向左扩展，只接受 `title_unit_allowed_chars` 中的字符
3. 取所有候选中最长的一个作为单位名

### 表头起始行（header_start_row=2）

手工做表的表头通常从第 2 行开始，占 3 行（多级合并表头）：
- 第 2–4 行：表头
- 第 5 行起：数据行
- 合计行：位于数据区末尾，首列包含「合计」

配置：`excel.header_start_row=2`，`excel.header_row_count=3`

### 额外汇总行

合计行之后的连续行中，首列匹配 `extra_summary_rows` 关键字的行会被解析：
- 溢缴款抵扣行：首列包含「溢缴款抵扣」
- 甲方转款行：首列包含「甲方转款」

程序会按与主表相同的 `column_indices` 从这些行提取各列数值。

---

## 校验公式

系统默认启用两项校验：

### 1. 横向公式（每行 + 合计行）

```
转款合计 = 扣款合计 + 个人所得税 + 个人其他调整 + 个人欠款 + 个人承担代理费 + 扣工会会费 - 交纳现金 + 实发工资 + 税费
```

### 2. 额外汇总校验

```
转款合计 = 溢缴款抵扣 + 甲方转款
```

**缺列处理**：如果某张工资表中缺少某些列（如扣款合计、个人所得税等），该列按 **0** 参与计算，公式仍然保持结构有效。例如司法局外包员工模板没有「个人所得税」列、但有「税费」列：个人所得税按 0 计、税费按实际值参与，「转款合计 = 扣款合计 + 实发工资 + 税费」的勾稽关系依然成立。

---

## 环境配置

从 [飞书开放平台](https://open.feishu.cn/) 获取以下配置：

| 配置项 | 获取路径 | 说明 |
|--------|----------|------|
| `FEISHU_APP_ID` | 应用开发 → 企业内部应用 → 应用信息 → App ID | 应用唯一标识 |
| `FEISHU_APP_SECRET` | 应用开发 → 企业内部应用 → 应用信息 → App Secret | 应用密钥 |
| `FEISHU_APPROVAL_CODE` | 飞书管理后台 → OA审批 → 审批流程 → 流程编码 | OA 审批流程唯一码 |

> **必须新建审批流程**：本项目的 13 字段表格结构与生产版 Payroll2Feishu 不同，建议基于生产流程复制一份，然后在「薪酬发放明细」表格控件中调整为 13 列，再填写新的 `FEISHU_APPROVAL_CODE`。

---

## 飞书权限配置

在 [飞书开放平台](https://open.feishu.cn/) → 应用开发 → 企业内部应用 → 权限管理 中开通：

| 权限点 | 用途 |
|--------|------|
| `contact:user.id:readonly` | 手机号查 open_id |
| `contact:user.department:readonly` | 查用户部门与姓名 |
| `approval:instance:write` | 创建审批实例 |
| `approval:form:readonly` | 读取审批模板字段（自动映射控件 ID） |
| `drive:file:write` | 上传审批附件 |

---

## 生产环境部署

### Systemd 服务

```ini
[Unit]
Description=Streamlit Payroll2Feishu Manual App
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/coding/submit_approval_feishu_for_manual/submit_approval_feishu_for_manual
ExecStart=/path/to/.venv/bin/streamlit run demo_app.py --server.port 8501 --server.address 0.0.0.0 --server.headless true
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Nginx 反向代理（可选）

```nginx
server {
    listen 80;
    server_name your-domain.com;
    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

---

## 配置文件指南

项目通过 `config.json` 自定义解析规则，无需修改代码即可适配不同工资表模板。

### 关键配置项

```json
{
  "excel": {
    "title_row": 1,
    "unit_name_row": 0,
    "header_start_row": 2,
    "header_row_count": 3,
    "summary_row_marker": "合计",
    "columns": {
      "transfer_total": {"keywords": ["转账合计", "转款合计"], "label": "转账合计(元)"},
      "deduction_total": {"keywords": ["扣款合计"], "label": "扣款合计(五险一金、单位代理费)"},
      "personal_tax": {"keywords": ["个人所得税", "个税"], "label": "个人所得税"},
      "adjustment": {"keywords": ["个人其他调整"], "label": "个人其他调整"},
      "personal_debt": {"keywords": ["个人欠款"], "label": "个人欠款"},
      "personal_proxy_fee": {"keywords": ["个人承担代理费"], "label": "个人承担代理费"},
      "union_fee": {"keywords": ["扣工会会费", "工会会费"], "label": "扣工会会费"},
      "pay_cash": {"keywords": ["交纳现金", "缴纳现金"], "label": "交纳现金"},
      "net_total": {"keywords": ["实发工资"], "label": "实发工资(元)"},
      "tax_and_fee": {"keywords": ["税费"], "label": "税费"}
    },
    "extra_summary_rows": {
      "overpayment_offset": {"keywords": ["溢缴款抵扣"], "label": "溢缴款抵扣"},
      "party_a_transfer": {"keywords": ["甲方转款"], "label": "甲方转款"}
    }
  },
  "validation": {
    "enabled": true,
    "strict": true,
    "tolerance": 0.00,
    "column_sum_checks": [
      {"column": "deduction_total"},
      {"column": "personal_tax"},
      {"column": "adjustment"},
      {"column": "pay_cash"},
      {"column": "net_total"},
      {"column": "tax_and_fee"}
    ],
    "row_formulas": [
      {
        "name": "转款合计 = 扣款合计 + 个人所得税 + 个人其他调整 + 个人欠款 + 个人承担代理费 + 扣工会会费 - 交纳现金 + 实发工资 + 税费",
        "lhs": "transfer_total",
        "rhs_plus": ["deduction_total", "personal_tax", "adjustment", "personal_debt", "personal_proxy_fee", "union_fee", "net_total", "tax_and_fee"],
        "rhs_minus": ["pay_cash"]
      }
    ],
    "extra_summary_checks": [
      {
        "name": "转款合计 = 溢缴款抵扣 + 甲方转款",
        "column": "transfer_total",
        "lhs_row_key": "summary",
        "rhs_plus_row_keys": ["overpayment_offset", "party_a_transfer"],
        "rhs_minus_row_keys": []
      }
    ]
  },
  "table_field": {
    "columns": [
      {"key": "report_name", "label": "报表名称"},
      {"key": "party_a_transfer", "label": "甲方转款"},
      {"key": "overpayment_offset", "label": "溢缴款抵扣"},
      {"key": "transfer_total", "label": "转账合计(元)"},
      {"key": "tax_and_fee", "label": "税费(元)"},
      {"key": "deduction_total", "label": "扣款合计(五险一金、单位代理费)"},
      {"key": "personal_tax", "label": "个人所得税"},
      {"key": "personal_debt", "label": "个人欠款"},
      {"key": "pay_cash", "label": "交纳现金"},
      {"key": "personal_proxy_fee", "label": "个人承担代理费"},
      {"key": "union_fee", "label": "扣工会会费"},
      {"key": "adjustment", "label": "个人其他调整"},
      {"key": "net_total", "label": "实发工资(元)"}
    ]
  },
  "ui": {
    "template_name": "工资发放审批(手工做表)",
    "description": "上传手工编写的工资发放表,自动校验转账合计/扣款合计/个税/实发工资等数值并提交飞书OA审批"
  },
  "feishu_form": {
    "approval_code_env": "FEISHU_APPROVAL_CODE",
    "controls": {
      "title": {"feishu_type": "input", "control_name_hint": "标题", "feishu_id": ""},
      "attachments": {"feishu_type": "attachmentV2", "control_name_hint": "附件", "feishu_id": ""},
      "table": {"feishu_type": "fieldList", "control_name_hint": "薪酬发放明细", "feishu_id": ""},
      "remark": {"feishu_type": "textarea", "control_name_hint": "备注", "feishu_id": ""}
    },
    "table_sub_controls": {
      "report_name": {"feishu_type": "input", "feishu_id": ""},
      "party_a_transfer": {"feishu_type": "amount", "feishu_id": ""},
      "overpayment_offset": {"feishu_type": "amount", "feishu_id": ""},
      "transfer_total": {"feishu_type": "amount", "feishu_id": ""},
      "tax_and_fee": {"feishu_type": "amount", "feishu_id": ""},
      "deduction_total": {"feishu_type": "amount", "feishu_id": ""},
      "personal_tax": {"feishu_type": "amount", "feishu_id": ""},
      "personal_debt": {"feishu_type": "amount", "feishu_id": ""},
      "pay_cash": {"feishu_type": "amount", "feishu_id": ""},
      "personal_proxy_fee": {"feishu_type": "amount", "feishu_id": ""},
      "union_fee": {"feishu_type": "amount", "feishu_id": ""},
      "adjustment": {"feishu_type": "amount", "feishu_id": ""},
      "net_total": {"feishu_type": "amount", "feishu_id": ""}
    }
  }
}
```

> **注意**：`feishu_form` 中的 `feishu_id` 默认为空字符串。首次启动时，若已配置 `FEISHU_APPROVAL_CODE`，程序会自动调用飞书 API 查询审批模板并尝试按标签匹配填充控件 ID。若自动匹配失败，需手动在飞书审批模板设计器中复制控件 ID 填入 `config.json`。

### 配置字段说明

`config.json` 共五个顶层字段：`excel`（解析规则）、`validation`（校验规则）、`table_field`（飞书表格字段）、`ui`（界面）、`feishu_form`（飞书控件映射）。

#### `excel` — 工资表解析规则

| 字段 | 含义 |
|------|------|
| `title_row` | 报表标题所在行（1 起，默认 1）。标题取该行第一个非空单元格，用作「报表名称」，并从中提取单位名与年月 |
| `unit_name_row` | 独立单位名称行（1 起）。`0` 表示没有独立行，改为从标题行提取单位名（默认 0，面向手工做表） |
| `header_start_row` | 表头起始行（1 起，默认 2） |
| `header_row_count` | 表头占用的行数，支持多级合并表头（默认 3，如「本月扣款 → 养老/失业/工伤/医疗 → 个人%/单位%」三级表头） |
| `summary_row_marker` | 合计行的首列标记（忽略所有空格，默认「合计」，故「合计」「合 计」都能识别） |
| `columns` | 列定义。键是程序内部使用的**列 ID**（`transfer_total`、`deduction_total`、`tax_and_fee` 等，也是校验公式引用的名字）；`keywords` 为表头匹配关键词（子串匹配，按数组顺序优先）；`label` 为界面显示名。**表中缺失的列在公式中按 0 参与计算** |
| `extra_summary_rows` | 合计行之后的额外汇总行（默认「溢缴款抵扣」「甲方转款」）。按首列关键词识别，数值按与主表相同的列位置提取 |
| `unit_name_patterns` | 从独立单位名称行提取单位名的正则（取 group 1） |
| `title_unit_patterns` | 从标题行提取单位名的正则（取 group 1，优先于后缀算法） |
| `title_unit_suffixes` / `title_unit_allowed_chars` | 无正则时，用「后缀枚举 + 向左贪婪扩展 + 取最长」从标题提取单位名的配置 |

#### `validation` — 数值校验规则

| 字段 | 含义 |
|------|------|
| `enabled` | 是否启用数值校验 |
| `strict` | 严格模式：校验未全部通过时**禁止提交**审批 |
| `tolerance` | 允许误差（元，默认 0.00）。浮点尾差场景可设 0.01 |
| `write_back_sheet` / `write_back_sheet_name` | 是否在原表末尾回写校验结果 sheet 及其名称（默认「验证结果」；.xls 只读不支持回写，会改为在汇总表里看明细） |
| `column_sum_checks` | **纵向加总**校验：对每个 `column`，求数据区各行该列之和，与合计行该列比较。只验证"行加总 = 合计行"，不验证列间关系 |
| `row_formulas` | **横向公式**校验：每条公式 `lhs` 列 = Σ`rhs_plus` 列 − Σ`rhs_minus` 列，**合计行与每个数据行分别校验**。缺列按 0 |
| `extra_summary_checks` | **额外汇总行**校验：`lhs_row_key` 为 `summary` 时取合计行，否则取额外汇总行（如 `party_a_transfer`）；`rhs_plus_row_keys` / `rhs_minus_row_keys` 为加/减项行，缺行不参与并提示 |
| `required_signatures` | 签名关键词检查（**读取顶层这份**）。扁平数组 `["A","B"]` 要求全部存在；分组数组 `[["A","B"],["C"]]` 时**任一组**全部命中即通过 |
| `rule_sets` | 规则集容器：每个规则集是一整套校验配置（`name`/`description`/`detect` + `column_sum_checks` + `row_formulas` + `extra_summary_checks`），用于适配不同模板 |
| `auto_detect` | 规则集自动匹配开关：`enabled=true` 且存在非 default 规则集时，按 `detect` 关键词对第一个附件打分选规则集；全部不命中回退 `default`（`fallback_to_default`） |

**顶层字段 vs `rule_sets` 的关系**：顶层 `column_sum_checks` / `row_formulas` / `extra_summary_checks` 是旧版扁平配置，作为兜底；**只要 `rule_sets.default` 存在，程序优先使用它**（`get_validation_rules` 逻辑）。修改默认规则时，两处应保持一致，防止 default 规则集被删后行为跳变。

**规则集 `detect` 自动匹配**（`detect_rule_set`）：每个规则集可配三个关键词组，命中一个 +1 分，得分最高者胜出，平分为 0 时回退 default：

| 关键词组 | 匹配对象 | 示例 |
|----------|----------|------|
| `title_keywords` | 报表标题（小写子串匹配） | `"外包员工"` → 命中「长春市司法局 2026年外包员工7月工资明细表」 |
| `unit_keywords` | 提取出的单位名（小写子串匹配） | `"司法局"` |
| `header_keywords` | **列的英文键名**（`column_indices` 的 key，不是中文表头！） | `"tax_and_fee"` |

#### `table_field.columns` — 飞书审批表格字段

13 个字段按顺序展示：`report_name`、`party_a_transfer`、`overpayment_offset`、`transfer_total`、`tax_and_fee`、`deduction_total`、`personal_tax`、`personal_debt`、`pay_cash`、`personal_proxy_fee`、`union_fee`、`adjustment`、`net_total`。`key` 对应解析/校验使用的列 ID，`label` 是飞书表格里的列名。

#### `ui` — 界面显示

| 字段 | 含义 |
|------|------|
| `template_name` | 页面标题（Streamlit `st.title` 显示用，**不是**审批标题；审批标题由 `generate_title()` 按「单位 + 年月 + 工资发放请示」格式生成） |
| `description` | 页面展示的项目描述 |

#### `feishu_form` — 飞书审批控件映射

| 字段 | 含义 |
|------|------|
| `approval_code_env` | 审批流程编码所取的环境变量名（默认 `FEISHU_APPROVAL_CODE`） |
| `controls` | 主表单控件：标题 / 附件 / 薪酬发放明细表 / 备注 |
| `table_sub_controls` | 明细表格内 13 列控件。`feishu_id` 留空时首次启动自动拉取审批模板控件 ID 按标签匹配填入；匹配失败需手动填写 |

---

## 已验证的 API 链路

| 步骤 | API | 说明 |
|------|-----|------|
| 1 | `contact/v3/users/batch_get_id` | 手机号 → open_id |
| 2 | `contact/v3/users/basic_batch` | open_id → 姓名 |
| 3 | `approval/openapi/v2/file/upload` | 上传附件 → file_code |
| 4 | `approval/v4/instances` | 创建审批实例 |
| 5 | `approval/v4/approvals/{approval_code}` | 查询模板字段（自动映射） |

---

## 技术栈

- Python 3.12
- Streamlit（Web UI）
- lark-oapi（飞书开放平台 SDK）
- openpyxl（Excel 解析）
- python-dotenv（配置管理）

---

## 双远程仓库

```bash
# GitHub
git remote add origin https://github.com/wanghannew1/Payroll2Feishu.git

# Gitee（国内镜像）
git remote add gitee https://gitee.com/wanghannew1/Payroll2Feishu.git
```

推送时同时推送到两个远端：
```bash
git push origin main && git push gitee main
```

---

## 作者

wanghannew1
