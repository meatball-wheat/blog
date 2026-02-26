---
layout: post
# remote_theme: mistakes/minimal-mistakes
title:  "快速在本地部署一个生成图片的模型"
date:   2026-02-26 17:00:00 +0800
# categories: memo
tags: 实践记录
---

#### 目的
最快速的方式在本地部署一个文生图模型，体验破产版nano banana ^_^

#### 环境
macOS，M3芯片（后面会有些特殊处理），本机已有git环境，而且安装了homebrew

#### 步骤和踩坑
**step1** 创建虚拟环境，安装各种依赖
```
# 创建专用环境（推荐）
python3 -m venv sd-env
source sd-env/bin/activate

# 安装核心库（自动适配Mac M系列）
pip install torch torchvision torchaudio  # 官方已内置MPS支持
pip install diffusers transformers accelerate safetensors Pillow
```

**step2**
本地测试使用的是sd1-5，因为模型比较小，下载快。由于网络环境问题，尝试了多种下载模型的方式，最终使用的是下面这种脚本直连的方式：
```python
#!/usr/bin/env python3
import os
import requests
from tqdm import tqdm
from pathlib import Path

# =============== 配置 ===============
#MODEL_ID = "runwayml/stable-diffusion-v1-5"  # 选用镜像站最全的模型
#LOCAL_DIR = "./models/sd1-5"                 # 本地保存路径
#MODEL_ID = "stabilityai/sd-turbo"
#LOCAL_DIR = "./models/sd-turbo"
MODEL_ID = "prompthero/openjourney-v4"
LOCAL_DIR = "./models/openjourney-v4"
# =====================================

# 关键文件列表（SD 1.5 必需组件）
REQUIRED_FILES = [
    "model_index.json",
    "scheduler/scheduler_config.json",
    "tokenizer/merges.txt",
    "tokenizer/special_tokens_map.json",
    "tokenizer/tokenizer_config.json",
    "tokenizer/vocab.json",
    "text_encoder/config.json",
    "text_encoder/pytorch_model.bin",
    "unet/config.json",
    "unet/diffusion_pytorch_model.safetensors",  # 核心权重（4.3GB）
    "vae/config.json",
    "vae/diffusion_pytorch_model.safetensors"
]

print(f"🌍 无认证直连下载: {MODEL_ID}")
print(f"📁 保存至: {os.path.abspath(LOCAL_DIR)}\n")

# 创建目录
Path(LOCAL_DIR).mkdir(parents=True, exist_ok=True)

# 逐个下载文件
for file_path in REQUIRED_FILES:
    # 构造直连URL（镜像站公开文件无需认证）
    url = f"https://hf-mirror.com/{MODEL_ID}/resolve/main/{file_path}"
    local_path = Path(LOCAL_DIR) / file_path
    local_path.parent.mkdir(parents=True, exist_ok=True)

    print(f"\n⬇️ 下载: {file_path}")
    print(f"🔗 URL: {url}")

    try:
        # 流式下载 + 进度条
        with requests.get(url, stream=True, timeout=60) as r:
            r.raise_for_status()
            total_size = int(r.headers.get('content-length', 0))

            with open(local_path, 'wb') as f, tqdm(
                total=total_size, unit='B', unit_scale=True,
                desc=f"  保存至: {local_path.name}"
            ) as bar:
                for chunk in r.iter_content(chunk_size=8192):
                    if chunk:
                        f.write(chunk)
                        bar.update(len(chunk))
        print(f"✅ 完成: {file_path}")

    except Exception as e:
        print(f"❌ 失败: {str(e)}")
        # 自动尝试备用URL格式
        alt_url = url.replace("/resolve/main/", "/blob/main/")
        print(f"🔄 尝试备用URL: {alt_url}")
        try:
            with requests.get(alt_url, stream=True, timeout=60) as r:
                r.raise_for_status()
                with open(local_path, 'wb') as f:
                    for chunk in r.iter_content(chunk_size=8192):
                        f.write(chunk)
            print(f"✅ 通过备用URL下载成功: {file_path}")
        except Exception as e2:
            print(f"❌ 备用URL也失败: {str(e2)}")
            print("💡 人工方案: 用浏览器打开URL → 点击'Download file'")
            continue

# 最终验证
print("\n✅ 下载完成！关键文件验证:")
core_file = Path(LOCAL_DIR) / "unet/diffusion_pytorch_model.safetensors"
if core_file.exists():
    size_gb = core_file.stat().st_size / 1e9
    print(f"  ✅ UNet权重: {size_gb:.2f} GB (正常范围: 4.2-4.4GB)")
else:
    print("  ❌ UNet权重缺失！请手动下载:")
    print(f"     https://hf-mirror.com/{MODEL_ID}/blob/main/unet/diffusion_pytorch_model.safetensors")

```

**step3** 创建generate脚本，加载模型和prompt，生成本地图片

```python
#!/usr/bin/env python3
import torch
import numpy as np
import os
import platform
from diffusers import StableDiffusionPipeline

# ===== 核心修复 1: 环境变量（MPS 必备）=====
os.environ['PYTORCH_ENABLE_MPS_FALLBACK'] = '1'  # 允许 MPS 不支持操作回退 CPU
os.environ['TOKENIZERS_PARALLELISM'] = 'false'

MODEL_PATH = "./models/sd1-5"
device = "mps" if torch.backends.mps.is_available() else "cpu"
print(f"✅ 设备: {device} | 模型路径: {os.path.abspath(MODEL_PATH)}")

# ===== 核心修复 2: 移除 variant + 强制 float32 =====
pipe = StableDiffusionPipeline.from_pretrained(
    MODEL_PATH,
    torch_dtype=torch.float32,  # MPS 唯一稳定选择
    use_safetensors=None,       # 自动回退 .bin
    local_files_only=True,
    safety_checker=None,
    requires_safety_checker=False,
    # ⚠️ 重要: 移除 variant="fp32"（仓库无此文件）
).to(device)

# ===== 核心修复 3: MPS 专属加固 =====
if device == "mps":
    print("🔧 应用 Apple Silicon 三重加固补丁")
    # 1. 禁用所有 slicing（MPS 兼容性问题源头）
    pipe.disable_attention_slicing()
    try:
        pipe.vae.disable_slicing()  # 新版 API
    except:
        pass

    # 2. 显式转换所有组件为 float32（防隐式降级）
    pipe.unet = pipe.unet.to(torch.float32)
    pipe.vae = pipe.vae.to(torch.float32)
    pipe.text_encoder = pipe.text_encoder.to(torch.float32)

    # 3. 修复 position_ids 警告（无害但干扰）
    if hasattr(pipe.text_encoder, 'embeddings') and hasattr(pipe.text_encoder.embeddings, 'position_ids'):
        pipe.text_encoder.embeddings.position_ids = torch.arange(
            77, dtype=torch.long, device=device
        ).unsqueeze(0)

# ===== 生成流程 =====
prompt = "a vibrant cyberpunk cat with glowing neon fur, detailed digital art, colorful background"
print(f"\n🎨 生成提示词: {prompt}")

# 固定种子 + 保守参数（MPS 生存法则）
generator = torch.Generator(device=device).manual_seed(1024)
try:
    with torch.no_grad():  # 显式禁用梯度（减少 MPS 不稳定）
        result = pipe(
            prompt,
            num_inference_steps=18,   # MPS 黄金步数（15-20）
            guidance_scale=4.5,       # 安全阈值（<5.0）
            generator=generator,
            output_type="pil"
        )

    image = result.images[0]

    # ===== 核心修复 4: 无效图像实时检测 =====
    img_array = np.array(image)
    if np.isnan(img_array).any() or img_array.mean() < 20:
        print("⚠️ 检测到无效图像（全黑/NaN）！应用终极修复重试...")
        # 二次尝试：更低引导强度 + 更少步数
        with torch.no_grad():
            result = pipe(
                prompt,
                num_inference_steps=12,
                guidance_scale=3.0,
                generator=generator,
                output_type="pil"
            )
        image = result.images[0]
        output = "cat_fixed.png"
    else:
        output = "cat.png"

    # 保存前确保 RGB 模式
    if image.mode != "RGB":
        image = image.convert("RGB")
    image.save(output)

    # 打印关键统计（验证有效性）
    stats = {
        "尺寸": image.size,
        "亮度均值": img_array.mean(),
        "最小值": img_array.min(),
        "最大值": img_array.max()
    }
    print(f"✨ 生成成功！图片: {output}")
    print(f"📊 验证数据: 尺寸={stats['尺寸']}, 亮度均值={stats['亮度均值']:.1f} (有效范围: 30-220)")

    # Mac 自动预览
    if platform.system() == "Darwin":
        os.system(f"open {output}")
        print("🖼️  已在预览中打开（请检查是否为彩色图像）")

except Exception as e:
    print(f"❌ 生成崩溃: {type(e).__name__}: {str(e)}")
    if "position_ids" in str(e):
        print("\n💡 位置ID修复已内置，此错误应已解决")
    elif "NaN" in str(e) or "nan" in str(e).lower():
        print("\n💡 MPS 终极建议:")
        print("1. 确保 PyTorch >= 2.1: pip install -U 'torch>=2.1.0' 'torchvision>=0.16.0'")
        print("2. 永远使用 torch.float32")
        print("3. guidance_scale 严格 ≤ 5.0")
```  

执行python generate.py即可 

#### 总结
整个过程完全通过和千问交互完成，要的困难都在于模型的下载过程。




<img src="{{ '/assets/images/cat.png' | relative_url }}" 
     alt="cat" 
     style="max-width: 50%; height: auto; display: block; margin: 0 auto;" />

