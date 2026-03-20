---
title: Gamma校正
created: 2025-03-19
updated: 2025-03-19
tags: [相机开发, ISP, 颜色校正, 基础, 显示技术]
---

# Gamma校正

## 定义

Gamma 校正是一种非线性亮度调整技术，用于补偿图像传感器线性响应与人眼非线性视觉感知之间的差异。通过幂函数变换，将线性 RAW 数据转换为符合人眼感知和显示设备特性的非线性数据，优化图像的亮度和对比度表现。

## 核心要点

### 为什么需要 Gamma 校正？

**人眼视觉特性**
- 人眼对暗部变化更敏感，对亮部变化相对不敏感
- 人眼感知近似对数/幂函数关系
- 需要更多位深存储暗部细节

**传感器线性响应**
- 传感器输出与光强成线性关系
- 若直接存储/显示，暗部细节丢失严重

**显示设备特性**
- 显示器具有固有 Gamma（通常 2.2）
- 需要进行逆变换以正确显示

### 数学原理

**Gamma 变换公式**
```
V_out = V_in ^ γ
```

- γ < 1：提亮图像，增强暗部细节
- γ = 1：线性变换
- γ > 1：压暗图像，增强亮部细节

**标准 sRGB Gamma**
- 整体近似 γ ≈ 2.2
- 暗部采用线性段（避免无限斜率）

```python
def srgb_gamma(v):
    """sRGB gamma 变换"""
    if v <= 0.0031308:
        return 12.92 * v
    else:
        return 1.055 * (v ** (1/2.4)) - 0.055

def srgb_gamma_inverse(v):
    """sRGB gamma 逆变换（线性化）"""
    if v <= 0.04045:
        return v / 12.92
    else:
        return ((v + 0.055) / 1.055) ** 2.4
```

### Gamma 曲线对比

```
亮度输出
  ↑
1 │                    ╱ sRGB (γ≈2.2)
  │                 ╱
  │              ╱
0.5│           ●─────── 线性 (γ=1.0)
  │        ╱
  │     ╱
  │  ╱  对数-like
  └──────────────────→ 亮度输入
  0        0.5       1
```

### 在 Pipeline 中的位置

```
... → [CCM] → [Gamma] → [CSC] → ...
              ↑
        在 RGB 域
        CCM 之后（线性空间校正完毕）
        色彩空间转换之前
```

**为什么在这个位置？**
- CCM 需要在线性空间计算
- Gamma 将线性数据转换为感知空间
- 后续降噪/增强在感知空间更有效

### Gamma 查找表 (LUT)

**硬件实现**
- 使用 LUT 加速计算
- 典型配置：64-256 个控制点
- 中间值通过线性插值

```c
// Gamma LUT 示例（简化）
const uint16_t gamma_lut[256] = {
    0,    12,   21,   29,   36,   43,   50,   57,
    // ... 更多值
    65535  // 最大值
};

// 应用 Gamma
output = gamma_lut[input >> 8];  // 查表
```

### 不同 Gamma 值效果

| Gamma | 效果 | 应用场景 |
|-------|------|----------|
| 1.8 | 较亮，暗部细节丰富 | 苹果早期显示器 |
| 2.2 | 标准，平衡 | sRGB、网页、视频 |
| 2.4 | 较暗，对比度高 | 专业摄影、印刷 |
| 2.6 | 更暗，电影感 | 数字电影 (DCI-P3) |

## 高级主题

### HDR 场景的 Tone Mapping

**问题**：HDR 图像动态范围超出显示设备能力

**解决方案**
- Global Tone Mapping：全局压缩动态范围
- Local Tone Mapping：局部自适应，保留细节

```python
def reinhard_tone_mapping(hdr_image):
    """Reinhard 全局色调映射"""
    L = luminance(hdr_image)
    L_white = max(L)  # 最大亮度
    L_tmo = L * (1 + L / (L_white ** 2)) / (1 + L)
    return apply_luminance(hdr_image, L_tmo)
```

### Gamma 与色彩空间

| 色彩空间 | Gamma | 应用 |
|----------|-------|------|
| sRGB | ≈2.2 | 通用显示、网页 |
| Rec.709 | ≈2.4 | 高清电视 |
| Rec.2020 | ≈2.4 | 超高清电视 |
| DCI-P3 | 2.6 | 数字电影 |
| Linear | 1.0 | 图像处理中间态 |

## 调试与测试

### 测试图卡
- **灰阶卡**：验证亮度层次
- **渐变图**：检查断层（Banding）
- **对比度图**：评估暗部/亮部细节

### 评价指标

| 指标 | 说明 | 目标 |
|------|------|------|
| **灰阶过渡** | 相邻灰阶可区分 | 无并阶 |
| **暗部细节** | 低亮度区域层次 | 可见 5% 灰阶 |
| **亮部细节** | 高亮度区域层次 | 可见 95% 灰阶 |

### 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 暗部断层 | Gamma 曲线过陡 | 减小 Gamma 值或增加位深 |
| 整体偏暗 | Gamma 值过大 | 调整 Gamma 为 2.2 |
| 暗部发灰 | Gamma 值过小 | 调整 Gamma 为 2.2 |
| 肤色不自然 | RGB 通道 Gamma 不一致 | 统一或微调各通道 Gamma |

## 相关概念

- [[ISP图像信号处理器]] - Gamma 是 RGB 域关键模块
- [[色彩校正矩阵]] - Gamma 的前置处理
- [[色彩空间]] - Gamma 与色彩空间紧密相关
- [[HDR]] - Gamma 在 HDR 中的扩展应用

## 参考来源

- [CSDN - ISP基础算法](https://blog.csdn.net/qq_40280673/article/details/140175744)
- [Espressif - Gamma Correction](https://deepwiki.com/espressif/esp-video-components/3.2-isp-pipeline)
- [arXiv - ISP Gamma Modeling](https://arxiv.org/pdf/2401.00151.pdf)
