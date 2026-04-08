# MetaClaw 安全审计报告

**审计日期**: 2026-04-08  
**审计版本**: v0.4.1.2  
**审计范围**: 核心代码 (metaclaw/) 及依赖

---

## 执行摘要

本次安全审计对 MetaClaw 项目进行了全面的安全分析，包括代码审查、依赖检查和潜在漏洞扫描。总体而言，项目代码质量良好，未发现严重的安全后门或恶意代码。发现的主要安全问题已修复。

---

## 审计结果概览

| 类别 | 状态 | 说明 |
|------|------|------|
| 恶意代码/后门 | ✅ 通过 | 未发现恶意代码或后门 |
| 命令注入 | ⚠️ 已修复 | 发现1处潜在命令注入风险，已添加安全验证 |
| 敏感信息泄露 | ✅ 通过 | API密钥和Token处理安全 |
| 依赖安全 | ✅ 通过 | 依赖项无已知严重漏洞 |
| 代码执行 | ✅ 通过 | 未发现eval/exec滥用 |

---

## 详细发现

### 1. 命令注入风险 (已修复) 🔧

**位置**: `metaclaw/openclaw_env_rollout.py:137`

**问题描述**:
`_exec_command()` 函数使用 `asyncio.create_subprocess_shell()` 执行 shell 命令，且命令参数来自 LLM 代理的工具调用，存在命令注入风险。

**原始代码**:
```python
async def _exec_command(cmd: str, timeout: float = 30.0) -> str:
    proc = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
```

**修复措施**:
已添加命令白名单验证机制：
- 只允许特定的安全命令: `openclaw`, `ls`, `cat`, `pwd`, `echo`, `head`, `tail`, `grep`, `done`
- 对命令进行解析和验证
- 拒绝执行不在白名单中的命令

**修复后代码**:
```python
async def _exec_command(cmd: str, timeout: float = 30.0) -> str:
    import shlex
    
    # Security: Validate command to prevent command injection
    ALLOWED_COMMANDS = {'openclaw', 'ls', 'cat', 'pwd', 'echo', 'head', 'tail', 'grep', 'done'}
    
    cmd_parts = shlex.split(cmd.strip())
    base_cmd = cmd_parts[0]
    
    if base_cmd not in ALLOWED_COMMANDS:
        return f"Error: command '{base_cmd}' is not allowed for security reasons"
    
    # ... rest of function
```

### 2. 敏感信息处理 (通过) ✅

**位置**: `metaclaw/auth_store.py`

**审计结果**:
- API密钥和OAuth Token存储在 `~/.metaclaw/auth-profiles.json`
- 凭证预览时只显示前12位和后4位，中间用 `...` 隐藏
- 支持Token过期检查和自动刷新
- 实现了错误退避机制，防止凭证被暴力破解

### 3. 子进程调用安全 (通过) ✅

**位置**: `metaclaw/api_server.py:1817`

**审计结果**:
- 使用 `asyncio.create_subprocess_exec()` 而非 `create_subprocess_shell()`
- 参数以列表形式传递，避免 shell 注入
- 设置了600秒超时，防止进程挂起
- 正确处理进程终止和错误

### 4. 依赖安全 (通过) ✅

**核心依赖**:
- `fastapi` / `uvicorn` - Web框架，无已知严重漏洞
- `httpx` - HTTP客户端，支持异步请求
- `torch` / `transformers` - ML框架，使用官方版本
- `click` - CLI框架，安全
- `pyyaml` - YAML解析，版本>=6.0安全

**建议**:
- 定期运行 `pip audit` 检查依赖漏洞
- 保持依赖项更新到最新版本

---

## 安全建议

### 高优先级

1. **命令白名单扩展**
   - 根据实际使用需求，审慎扩展 `_exec_command()` 的允许命令列表
   - 考虑添加更细粒度的命令参数验证

2. **输入验证强化**
   - 对所有外部输入（用户消息、配置文件）进行严格的输入验证
   - 实施长度限制和字符白名单

### 中优先级

3. **日志安全**
   - 确保日志中不包含敏感信息（API密钥、Token等）
   - 当前实现已较好，建议定期审查

4. **文件权限**
   - 确保 `~/.metaclaw/` 目录权限设置为 `700`
   - 配置文件应设置为 `600`

### 低优先级

5. **依赖监控**
   - 建立依赖更新自动化流程
   - 订阅相关安全公告

---

## 后门检查

### 检查项目

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 网络连接 | ✅ 正常 | 仅连接配置的LLM API和Tinker服务 |
| 文件系统访问 | ✅ 正常 | 仅访问配置目录和临时文件 |
| 进程创建 | ✅ 正常 | 仅创建必要的子进程 |
| 代码执行 | ✅ 正常 | 无动态代码执行 |
| 数据外泄 | ✅ 正常 | 无异常网络传输 |

### 代码审查结论

经过全面代码审查，未发现以下恶意行为：
- 隐藏的网络连接
- 数据窃取代码
- 远程控制后门
- 加密货币挖矿代码
- 键盘记录器
- 文件加密勒索代码

---

## 修复记录

| 日期 | 问题 | 修复文件 | 修复内容 |
|------|------|----------|----------|
| 2026-04-08 | 命令注入风险 | `metaclaw/openclaw_env_rollout.py` | 添加命令白名单验证 |

---

## 结论

MetaClaw v0.4.1.2 经过安全审计后，代码整体安全状况良好。发现的命令注入风险已修复。建议定期（每季度）进行安全审计，并在重大版本更新时进行专项安全审查。

---

**审计人员**: AI Security Assistant  
**报告生成时间**: 2026-04-08
