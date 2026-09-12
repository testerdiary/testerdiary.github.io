---
title: "测试用例会腐化，得像代码一样治理：testerbuddy × MeterSphere 三件套实践"
date: 2026-09-12 14:40:00 +0800
categories: [测试相关]
tags: [MeterSphere, 测试设计, 测试治理, 用例腐化, Cockburn, 质量门禁]
---

测试用例会腐化：需求改版后步骤停在上一版，同一场景在三个模块各躺一份，偶尔一次"探测"就能把整棵模块树连带用例删掉，回收站里连影子都没有。用例是资产，不是一次性产出，它需要像代码一样被设计、被治理、被追溯。testerbuddy 把这条纪律做成 Codex skill，落在 MeterSphere 的模块树与功能用例上：design 从需求生成用例树，case-decay 治理已经腐化的用例，code-case-link 让代码和用例互相找得到，三件套外面还罩着一套事故换来的硬纪律：先 dry-run，再确认，最后才动手。

## 腐化有三个信号，最狠的那个一次 GET 就清零

需求改了用例没改，是慢性腐化：步骤还停在旧逻辑，预期结果与现状对不上。同一场景被复制到多个模块，是结构性腐化：搜索时看起来每条都在，跑的时候每份都可能过时。误删是急性腐化：一个删除接口收到 GET 也会执行，级联删掉整棵子树，回收站不留记录。

| 信号 | 典型症状 | testerbuddy 的对应处理 |
| --- | --- | --- |
| 过期步骤 | 需求改版，步骤与预期还停在旧逻辑 | design 审计出需求缺口；decay 诊断用证据逐条对比 |
| 重复用例 | 同一场景散落多个模块 | 生成期 deduplicator 去重；decay 阶段合并 |
| 误删 | GET 级联删除且不进回收站 | 非空模块拒绝删除；删除必须显式 `--yes` |

最危险的是第三种，所以脚本先做了防御：模块删除只允许空模块（`count=0` 且无子模块），非空模块直接拒绝，真想清空就降级为重命名、加上 `[空-待删除]` 前缀，留给人确认：

```bash
python <SKILL_DIR>/scripts/delete_cases.py \
  --ids uid1,uid2 --type module --dry-run --yes
```

## 先立边界再谈生成：需求之外不猜，缺的写进澄清单

Cockburn-Deng 六条纪律里，真正拦住幻觉的是两条：证据边界和零猜测。设计只吃用户提供的需求文本和本地文件，不联网、不猜系统行为；actor、前置条件、失败路径缺失，就写进"问题与澄清"清单，用例挂上 `gap_ref` 标注"待澄清"，而不是替系统编一个响应。

落库之前还要过 9 项质量门禁，逐项自检，阻塞项不补齐就不许落库：

```text
# testbuddy-design/references/qa/quality-gate.md（9 项）
# 1  Scope 明确              2  Sea-Level（用户目标级）
# 3  前置条件可测            4  MSS 主动主谓宾（无 UI 细节）
# 5  成功保证可验证          6  最小保证
# 7  扩展编码                8  证据边界
# 9  不可测定性词已移除/量化
```

## design：需求到用例树，中间隔着一道 dry-run 的门

生成器把需求变成 FEATURE/SCENE/TEST_POINT/CASE 四层设计树：FEATURE 建一级模块，SCENE 建二级模块，TEST_POINT 不建模块、只作为用例名前缀 `{测试点}::`，CASE 落成功能用例，tags 带上 FEATURE 和 SCENE 的名字：

```json
{
  "nodes": [
    { "uid": "f1", "name": "优惠券结算", "kind": "FEATURE", "parent_uid": "root" },
    { "uid": "s1", "name": "结算主流程", "kind": "SCENE", "parent_uid": "f1" },
    { "uid": "c1", "name": "抵扣校验", "kind": "TEST_POINT", "parent_uid": "s1" },
    { "uid": "case1", "name": "叠加多张券取最优", "kind": "CASE", "parent_uid": "c1",
      "instance": {
        "preconditions": "已登录且购物车含可结算商品",
        "priority": "P1",
        "steps": [
          { "action": "用户提交结算，携带多张可用优惠券", "expected": "系统选择最优抵扣并返回应付金额" }
        ]
      }
    }
  ]
}
```

落库不是一条命令的事：先 `--dry-run` 打印将要创建的模块和用例清单，人确认后才正式执行；模块归属也由人决定，先列出现有模块树，选择复用/挂载已有模块，还是新建 `{需求名}_{yyyyMMdd}` 根模块：

```bash
python <SKILL_DIR>/scripts/ms_design.py --design draft.json \
  --name 优惠券结算_20260912 --parent root --dry-run
```

## case-decay：治理先出报告，等你回复"确认执行"

腐化治理不是上来就删。流程是先拉取目标模块的用例，按诊断规则逐条比对证据，生成 HTML 确认报告，等用户回复"确认执行"，再按 move → update → delete → rename → 清理空模块的顺序动手，每一步仍然先 dry-run。诊断依据可追溯：删什么、为什么删，报告里都有出处，不是模型说了算。

## code-case-link：没有映射，改代码就是盲改

代码和用例之间需要一张显式映射表，写在 `.testbuddy/code_case_relations.json`：

```json
{
  "1234567890": {
    "file": "tests/test_login.py",
    "function": "TestLogin.test_success"
  }
}
```

```bash
python <SKILL_DIR>/scripts/record_code_case_relation.py \
  --case 1234567890 --file tests/test_login.py --function TestLogin.test_success
```

需求变更时反查这张表，能立刻圈出会被波及的用例和断言；自动化跑完也能回来对账：这条用例到底有没有代码覆盖。

## 一次 GET 事故，换来三条硬纪律

2026-09-04，一次"只读探测"对删除接口发了个 GET。`GET /functional/case/module/delete/{id}` 收到 GET 就直接执行删除，级联清掉一整棵模块树和下面的用例，回收站里只有 383 条历史用例，跟这个子树毫无关系，一条都找不回来。

```bash
# 事故现场（2026-09-04）：GET 也会执行删除，且不进回收站
curl "<BASE_URL>/functional/case/module/delete/{module_id}"
```

事故之后，技能包写死了三条纪律：

1. 语义为删除/变更的接口禁止用 GET 探测，即使"只读"也不行；接口是否存在，只靠已知只读接口和 405/404 来区分。
2. 所有变更操作先 `--dry-run` 出清单，验证限定在 `testerbuddy-SMOKE-` 前缀的沙箱模块，禁止直接在真实业务树上试。
3. 删除类操作必须显式 `--yes`，且只针对指定目标；级联删除更要二次确认。

这套纪律的代价是慢：每一步写操作都要人点头，自动化程度远不如"一键生成"。但测试资产删光了不会自动长回来，慢即是快。

## 结论：工具管流程，纪律管灾难

testerbuddy 的价值不在脚本，在脚本背后的门槛：需求之外不猜、落库前过门禁、删除前确认、代码与用例对账。这套"dry-run + 显式确认 + 沙箱验证"的流程与平台无关，换到任何测试平台，同样的纪律都成立。
