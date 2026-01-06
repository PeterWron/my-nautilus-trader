# Google Colab 一键配置 Nautilus Trader 环境指南 (v5 - 兼容性修复版)

这个版本解决了 `ResolutionImpossible` 依赖冲突问题，并优化了安装流程。

## 1. 环境配置代码

请在 Colab 中运行以下代码块。

**运行逻辑说明：**
1. **第一次运行**：脚本会安装 `nautilus_trader` 及其核心依赖，然后**自动重启 Colab 运行时**。
2. **第二次运行**：重启后，请**再次点击运行**同一个单元格，脚本将完成代码修复和仓库克隆。

```python
import os
import sys
import shutil
import re

# --- 1. 自动安装与重启逻辑 ---
try:
    import nautilus_trader
    from nautilus_trader.core.data import Data
    print("✅ Nautilus Trader 核心模块已成功加载。")
except (ImportError, ModuleNotFoundError):
    print("⏳ 正在安装依赖并配置环境，请稍候...")
    # 移除冲突的 [visualization] 额外依赖，改为直接安装核心包
    # 不再强制锁定 pandas 版本，让 pip 自动协调
    !pip install -q "nautilus_trader[polymarket]" "plotly" "matplotlib"
    print("\n🔄 安装完成！正在自动重启 Colab 运行时以加载新模块...")
    import os
    os.kill(os.getpid(), 9)

# --- 2. 克隆仓库 (仅在重启后执行) ---
if not os.path.exists("my-nautilus-trader"):
    print("⏳ 正在克隆 GitHub 仓库...")
    !git clone -q -b develop https://github.com/PeterWron/my-nautilus-trader.git
%cd my-nautilus-trader

# --- 3. 应用 Polymarket 兼容性修复 ---
def patch_nautilus():
    import nautilus_trader
    base_path = os.path.dirname(nautilus_trader.__file__)
    
    # 同步仓库中的适配器代码
    src_adapter = "nautilus_trader/adapters/polymarket"
    dst_adapter = os.path.join(base_path, "adapters/polymarket")
    
    if os.path.exists(src_adapter):
        if os.path.exists(dst_adapter):
            shutil.rmtree(dst_adapter)
        shutil.copytree(src_adapter, dst_adapter)
        print(f"✅ 已同步适配器代码到: {dst_adapter}")

    package_path = os.path.join(dst_adapter, "loaders.py")
    if not os.path.exists(package_path):
        print(f"❌ 未找到目标文件: {package_path}")
        return

    with open(package_path, "r") as f:
        content = f.read()

    # 注入缺失的导入和辅助函数
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

    # 修复时间戳转换
    if "start_time_s = start_time_ms // 1000" not in content:
        content = content.replace("all_snapshots = []", "all_snapshots = []\n        start_time_s = start_time_ms // 1000\n        end_time_s = end_time_ms // 1000")
    
    content = content.replace("\"startTs\": start_time_ms", "\"startTs\": start_time_s")
    content = content.replace("\"endTs\": end_time_ms", "\"endTs\": end_time_s")

    with open(package_path, "w") as f:
        f.write(content)
    print("✅ Nautilus Trader 环境修复完成！")

patch_nautilus()
```

## 2. 运行示例

```python
# 运行示例脚本
!python examples/backtest/polymarket_simple_quoter.py
```
