# 这个 fork 改了什么

上游：[bybit-exchange/svg-diagram](https://github.com/bybit-exchange/svg-diagram)
本 fork：`mondkartoffeln/svg-diagram`

改动**只有两处**，都在 `SKILL.md`，共 **+30 行、0 删除**。
随时可以用 `git diff upstream/main` 复核。

---

## 改动一：补 `whenToUse` 字段

```yaml
whenToUse: The user asks for an architecture diagram, flowchart, sequence/lifecycle/data-flow
  figure, or any SVG illustration for a document or README; ...
```

★ 原因：原版只有 `name` + `description`（Claude Code 的格式）。
[DSH](https://github.com/deepseek-ai) 的技能约定是 **`name` + `description` + `whenToUse`** 三个字段。
实测少了它也能被加载（字段可选），但补上能让技能路由更准。

---

## 改动二：让 Agent 真的去跑 linter（**这是主要目的**）

在 `## Verification checklist` 前面插入一段强制指令，要点三条：

1. **先跑 linter，别靠肉眼走清单**：
   ```bash
   node tools/svg-lint/bin/svg-lint.mjs path/to/diagram.svg
   ```
2. ★ **达标条件是「exit 0 **且** 没有 `WARNINGS PRESENT`」** —— 不能只看退出码
3. 人工清单只保留 linter 判不了的那几条（连接线的**含义**是否可推断、虚线分组框样式是否统一）

### 为什么加这一段

**原版 `SKILL.md` 里 `grep "lint"` 命中数为 0。**

也就是说：上游把一个 **12 个检查器、984 个测试** 的 linter 带在仓库里，
但**技能正文一次都没让 Agent 去跑它** —— 只有 `README.md`（给人看的）提到了。

结果是 Agent 仍然靠「自己走一遍人工清单」，而**那份清单里几乎每一条，linter 里都已经实现了**：

| 人工清单（靠自觉） | linter 检查器 |
|---|---|
| No overlapping elements | `checks/overlap.mjs` |
| No text right edge intrudes | `checks/text-overflow.mjs` |
| Markers declare `markerUnits` | `checks/arrow-marker.mjs` |
| Connectors use C/Q curves | `checks/connector-geometry.mjs` |
| Block spacing ≥25px | `checks/block-spacing.mjs` |
| viewBox matches content | `checks/viewbox-clipping.mjs` |
| Box heights sufficient | `checks/box-height.mjs` |
| Special characters escaped | `checks/xml-escaping.mjs` |

**自带检查器却不用，等于把钱花在抽屉里。**

### 关于 exit code 的坑（值得单独记）

CLI 的契约是：

| 退出码 | 含义 |
|---|---|
| `0` | **没有 error** —— warnings **可能仍然存在** |
| `1` | 至少一个 error |

★ 所以「exit 0」**不等于**过。房屋标准是 **0 errors 且 0 warnings**，
有 warning 时文本报告末尾会多一行 `!! WARNINGS PRESENT`。

★★ 我第一版补丁就写错成「re-run until it exits 0」——**查了上游 README 才改对**。
（`tools/svg-lint/README.md` 第 57 行写得很清楚。）

---

## 从上游同步

```bash
git fetch upstream
git diff upstream/main              # 看上游新变化
git merge upstream/main             # 合进来（本 fork 只动 SKILL.md，几乎不会冲突）
git push origin main
```

★ 如果上游哪天自己把 linter 接进 `SKILL.md` 了，**改动二就可以丢掉** —— 这正是做成 fork 的意义。

---

## 装到 DSH

```bash
git clone https://github.com/mondkartoffeln/svg-diagram.git ~/.dsh/skills/svg-diagram
```

★ DSH **自动扫描** `~/.dsh/skills/`，无需重启、无需注册。

---

## 验证记录

本 fork 的内容在上游代码上实测通过：

| 验证 | 结果 |
|---|---|
| linter 自带测试套件 | **tests 984 · pass 984 · fail 0** |
| 上游 16 个 fixture（14 fail + 2 pass） | **16/16 符合预期**（按「exit 0 且无 WARNINGS PRESENT」判） |
| 10 个 gallery 示例 | **10/10 干净** |
| 端到端闭环：造错图 → lint → 按 repair 改 → 再 lint | **走到 0 error 0 warning** |

★ 端到端那步最能说明价值：我「认真写的」第二版**肉眼完全正常**，
但 linter 量出 **边距上下差 2.5px、箭头两端间隙 0px（该 5px）** ——
**这正是「放进 README 就有点尴尬」的那类问题。**
