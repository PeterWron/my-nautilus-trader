# Google Colab 一键配置 Nautilus Trader 环境指南 (v4 - 终极修复版)

这个版本解决了 `ModuleNotFoundError`，并实现了安装后的自动重启逻辑。

## 1. 环境配置代码

请在 Colab 中运行以下代码块。

**运行逻辑说明：**
1. **第一次运行**：脚本会检测是否安装了 `nautilus_trader`。如果没有，它会执行安装，然后**自动重启 Colab 运行时**（您会看到单元格执行中断，这是正常的）。
2. **第二次运行**：重启后，请**再次点击运行**同一个单元格。此时脚本会检测到已安装，并自动执行代码修复和仓库克隆逻辑。

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
    # 锁定版本以确保兼容性
    !pip install -q "pandas==2.2.2" "nautilus_trader[polymarket,visualization]"
    print("\n🔄 安装完成！正在自动重启 Colab 运行时以加载新模块...")
    # 自动重启 Colab 运行时
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

## 3. 常见问题
- **为什么单元格运行到一半停止了？**
  这是因为脚本在安装完 `nautilus_trader` 后，必须通过杀死当前进程来强制 Colab 重启运行时，这样才能加载新安装的二进制扩展。请在停止后**再次点击运行**即可。
- **为什么还是提示 ModuleNotFoundError？**
  请确保您在重启后再次运行了该单元格。如果问题依旧，请尝试点击 Colab 菜单栏的 `Runtime` -> `Disconnect and delete runtime`，然后重新开始。
