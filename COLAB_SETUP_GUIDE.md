# Google Colab 一键配置 Nautilus Trader 环境指南 (v3 - 修复版)

针对您遇到的 `ModuleNotFoundError` 和 `pandas` 冲突问题，我优化了安装流程。

## 1. 环境配置代码

请在 Colab 中运行以下代码块。**注意：安装完成后，Colab 可能会提示您“RESTART SESSION”，请点击该按钮，然后再次运行此单元格（第二次运行会跳过安装直接执行修复逻辑）。**

```python
import os
import sys
import shutil
import re

# --- 1. 安装依赖 (仅在未安装时执行) ---
try:
    import nautilus_trader
    print("✅ Nautilus Trader 已安装。")
except ImportError:
    print("⏳ 正在安装依赖，请稍候...")
    # 锁定 pandas 版本以兼容 Colab，并安装核心扩展
    !pip install -U "pandas==2.2.2" "nautilus_trader[polymarket,visualization]"
    print("⚠️ 安装完成！请点击页面下方的 'RESTART SESSION' 按钮，然后重新运行此单元格。")
    # 强制停止当前执行，提醒用户重启
    sys.exit()

# --- 2. 克隆仓库 ---
if not os.path.exists("my-nautilus-trader"):
    !git clone -b develop https://github.com/PeterWron/my-nautilus-trader.git
%cd my-nautilus-trader

# --- 3. 应用修复逻辑 ---
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

## 3. 为什么会出现之前的错误？
1. **ModuleNotFoundError**: Nautilus Trader 使用 Rust 编写了核心组件。在 Colab 中通过 `pip` 安装后，Python 的当前进程无法立即加载新安装的动态链接库（.so 文件）。必须**重启 Session** 才能让 Python 重新扫描并加载这些模块。
2. **Pandas 冲突**: Colab 预装了 `pandas 2.2.2`，而新版 Nautilus 默认会尝试安装最新版。锁定版本可以避免环境不稳定的警告。
