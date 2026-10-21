# 产品知识问答系统测试记录

2026-10-21，macOS 26.6.2 + Python 3.12.10。

## 先说结果

核心流程都跑通了：33 个自动化测试全过，8 个固定评测样例全过，启动服务后 46 个接口检查也全过。HTML 和 DOCX 导入正常。发现两个小问题，建议交付前处理一下。

测试用的是临时 SQLite 库，没碰正式库。测试前后 `data/rag.sqlite3` 都是 16 个产品、17 份资料、51 条 FAQ。

## 怎么测的

环境按 `requirements.txt` 装，实际用到的是 FastAPI 0.157.0、Starlette 1.9.0、uvicorn 0.54.0、scikit-learn 1.9.1，测试脚本另外装了 httpx。

先跑 `tests/` 下的 33 个用例，再跑 `evaluate.py` 的 8 个固定样例。然后把 uvicorn 起在 8123 端口，用临时脚本对 18 个接口发了 46 个请求，正常流程、边界值和错误输入都试了。最后补了 HTML/DOCX 导入和 macOS 启动脚本的检查。

## 结果

- 自动化测试 33/33，耗时 2.8 秒左右。
- 固定评测 8/8，状态、答案关键字、来源命中率都是 1.0。
- 接口检查 46/46，没有 5xx，服务日志也没报错。
- HTML 样例导入后生成 100 个片段、6 条 FAQ。
- DOCX 导入后生成 3 个片段、2 条 FAQ，导入完能直接回答。

重点看过的流程包括：2.0/1.0 版本隔离、资料不足拒答、退款转人工、多轮追问、预览不写库、导入后马上能问、手工补 FAQ、知识缺口补答案、反馈、评估不污染正式数据、删除文档后不再命中。

顺便记了下响应时间：首页大概 2.4 ms，一次问答端到端约 4 ms，8 个样例的评估接口约 0.6 秒。都是单机单用户的数字，不代表并发情况。

## 发现的问题

### 1. 省略布尔参数时，清洗默认值没生效

文档预览和上传接口里，`remove_page_markers` 和 `remove_duplicates` 默认写的是 `True`，但不传这两个字段时，返回的设置是：

```json
{"remove_page_markers": false, "remove_duplicates": false}
```

结果就是页码行和相邻重复行都保留了下来。显式传 `true` 时是正常的。

我拿一个最小的 FastAPI 应用试了同样的写法，`bool = Form(True)` 省略字段时拿到的是 `False`，只有显式传 `"true"` 才变回 `True`。当前装的是 FastAPI 0.157.0 + Starlette 1.9.0，看起来是这两个版本的行为，不是清洗函数本身的问题。

网页端暂时不受影响，因为前端提交时会把两个复选框的状态显式带上，而且默认是勾选的。会踩到这个坑的是直接调 API、用 `/docs` 调试，或者第三方对接的人。

改法可以二选一：把 FastAPI/Starlette 固定到行为正常的版本，或者在接口里显式处理这两个字段。另外建议补一条“不传这两个参数”的测试，现在的用例都是显式传值的。

### 2. start_mac.command 没有可执行权限

按 README 运行 `./start_mac.command` 会直接失败：

```text
zsh: permission denied: ./start_mac.command
```

退出码是 126。改成 `bash start_mac.command --setup-only` 就能正常走完环境检查、依赖校验和临时库初始化。

改法很简单，执行 `chmod +x start_mac.command`，提交时把可执行位一起带上就行。

### 3. 测试时有个弃用警告

跑 `unittest` 会提示：

```text
StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
```

不影响这次结果，33 个用例都通过了。以后升级依赖时顺手迁到推荐的测试客户端就行。

## 没测到的部分

- **PDF 导入**：只实际试了 Markdown、HTML 和 DOCX，PDF 没造样例。
- **Windows 实机启动**：这次只在 macOS 上跑，相关脚本只做了静态检查。
- **可选 LLM**：没配置模型参数，测的是不调用模型时的离线回答。
- **浏览器端到端**：页面能打开，前后端字段也对过，但没有用浏览器实际点一遍。
- **并发和压力**：没有做并发上传、长会话或大知识库的测试。
- **权限**：服务本身没有鉴权，能访问到端口的人都能上传、删除资料或改 Prompt，只适合本机用。

## 最后

整体可以正常演示：问答、版本隔离、拒答、转人工、文档导入、人工补充、反馈和评估都没问题。两个问题里，第 1 个建议优先修，尤其是还要用 API 调试或对接的话；第 2 个改一下权限就好。

测试服务已经停了，测试脚本和日志在 `/tmp`，没有放进仓库。

常用命令：

```bash
RAG_DB=/tmp/rag-test.sqlite3 .venv/bin/python -m unittest discover -s tests -v
.venv/bin/python evaluate.py
```
