# Google Colab 一键配置 Nautilus Trader 环境指南 (v2)

您可以在 Google Colab 中新建一个 Notebook，并运行以下代码块。这段代码将自动完成仓库克隆、依赖安装以及环境修复。

## 1. 环境配置代码

请将以下代码复制到 Colab 的第一个单元格中运行：

```python
# 1. 克隆仓库 (使用最新的 develop 分支)
!git clone -b develop https://github.com/PeterWron/my-nautilus-trader.git
%cd my-nautilus-trader

# 2. 安装核心依赖及 Polymarket 扩展
!pip install -U "nautilus_trader[polymarket,visualization]"

# 3. 应用 Polymarket 兼容性修复脚本
import os
import re
import site
import shutil

def patch_nautilus():
    # 获取已安装包的路径
    import nautilus_trader
    base_path = os.path.dirname(nautilus_trader.__file__)
    
    # 同步仓库中的适配器代码到已安装的包中
    src_adapter = "nautilus_trader/adapters/polymarket"
    dst_adapter = os.path.join(base_path, "adapters/polymarket")
    
    if os.path.exists(src_adapter):
        shutil.copytree(src_adapter, dst_adapter, dirs_exist_ok=True)
        print("✅ 已同步仓库中的 Polymarket 适配器代码。")

    package_path = os.path.join(dst_adapter, "loaders.py")
    if not os.path.exists(package_path):
        print(f"❌ 未找到目标文件: {package_path}")
        return

    with open(package_path, "r") as f:
        content = f.read()

    # 注入 HttpMethod 和辅助函数
    if "from nautilus_trader.core.nautilus_pyo3.network import HttpMethod" not in content:
        patch = """from __future__ import annotations
from nautilus_trader.core.nautilus_pyo3.network import HttpMethod
def _build_url(url, params):
    if not params: return url
    from urllib.parse import urlencode
    query = urlencode(params)
    return f"{url}?{query}" if "?" not in url else f"{url}&{query}"
"""
        content = re.sub(r"from __future__ import annotations", patch, content)

    # 修复 HttpClient 调用
    content = content.replace("client.get(", "client.request(HttpMethod.GET, ")
    content = content.replace("self._http_client.get(", "self._http_client.request(HttpMethod.GET, ")
    
    # 修复 params 传参方式
    content = re.sub(r"await (.*)\.request\(HttpMethod\.GET,\s*url=(.*),\s*params=(.*)\s*\)", 
                     r"await \1.request(HttpMethod.GET, url=_build_url(\2, \3))", content)

    # 修复时间戳转换 (针对 load_orderbook_snapshots 和 load_trades)
    content = content.replace("start_time_ms = int(start.timestamp() * 1000)", 
                             "start_time_ms = int(start.timestamp() * 1000)\n        start_time_s = start_time_ms // 1000")
    content = content.replace("end_time_ms = int(end.timestamp() * 1000)", 
                             "end_time_ms = int(end.timestamp() * 1000)\n        end_time_s = end_time_ms // 1000")
    
    # 针对 fetch_orderbook_history 内部定义变量
    if "start_time_s = start_time_ms // 1000" not in content:
         content = content.replace("all_snapshots = []", "all_snapshots = []\n        start_time_s = start_time_ms // 1000\n        end_time_s = end_time_ms // 1000")

    # 替换为秒级时间戳
    content = content.replace("\"startTs\": start_time_ms", "\"startTs\": start_time_s")
    content = content.replace("\"endTs\": end_time_ms", "\"endTs\": end_time_s")

    with open(package_path, "w") as f:
        f.write(content)
    print("✅ Nautilus Trader 环境修复完成！")

patch_nautilus()
```

## 2. 运行示例代码

配置完成后，您可以在下一个单元格中运行 Polymarket 示例：

```python
import asyncio
import os

# 运行示例脚本 (注意：如果报错 No historical data，请尝试修改脚本中的 lookback_hours)
!python examples/backtest/polymarket_simple_quoter.py
```

## 3. 注意事项
- **数据可用性**：Polymarket 的历史数据接口有时会因为时间范围过大或市场不活跃而返回空数据。如果看到 `ValueError: No historical data available`，这是 API 返回了空结果，而非代码错误。
- **Token 安全**：请务必撤销您之前提供的 GitHub Token。
