# Banner Image Lazy Loading Optimization

## 需求概述
优化 banner 图片加载机制，实现按需加载而非预加载所有资源，提高页面性能。

## 问题分析

### 旧实现存在的问题：
1. **无论轮播是否启用，所有banner图片都会被渲染到HTML中**
2. **随机模式下（carousel.enable: false）**：虽然只显示一张，但所有6张图片都被加载
3. **轮播模式下（carousel.enable: true）**：所有图片一次性加载，阻塞首屏渲染
4. **没有使用lazy loading属性**：所有图片都立即加载
5. **全屏壁纸组件存在同样问题**

## 优化方案

### 1. 服务端随机选择（Random Mode）
**位置**: `src/layouts/MainGridLayout.astro`

```typescript
// 新增：在服务端判断是否启用轮播，决定加载策略
const isCarouselEnabled = siteConfig.banner.carousel?.enable;
const desktopImagesArray = Array.isArray(bannerImages.desktop) ? bannerImages.desktop : [bannerImages.desktop];
const mobileImagesArray = Array.isArray(bannerImages.mobile) ? bannerImages.mobile : [bannerImages.mobile];

// 如果不启用轮播且有多张图片，随机选择一张
let selectedDesktopImages = desktopImagesArray;
let selectedMobileImages = mobileImagesArray;

if (!isCarouselEnabled && desktopImagesArray.length > 1) {
    const randomIndex = Math.floor(Math.random() * desktopImagesArray.length);
    selectedDesktopImages = [desktopImagesArray[randomIndex]];
    selectedMobileImages = [mobileImagesArray[randomIndex] || mobileImagesArray[0]];
}
```

**好处**：
- 随机模式下只渲染1张图片到HTML，其他5张不加载
- 减少约83%的图片请求（6张减少到1张）

### 2. 智能Lazy Loading（Carousel Mode）
**位置**: `src/layouts/MainGridLayout.astro` banner渲染部分

```astro
{selectedDesktopImages.map((src, index) => {
    // 第一张图片立即加载，其他图片延迟加载
    const shouldLazyLoad = index > 0;
    
    return (
        <li class={`carousel-item ...`}>
            <ImageWrapper 
                src={src} 
                loading={shouldLazyLoad ? "lazy" : "eager"}
            />
        </li>
    );
})}
```

**好处**：
- 第一张图片 `loading="eager"` 立即加载（优化LCP）
- 其他图片 `loading="lazy"` 浏览器原生延迟加载
- 图片会在即将进入视口时才加载

### 3. ImageWrapper组件升级
**位置**: `src/components/misc/ImageWrapper.astro`

```typescript
interface Props {
    // ... 其他属性
    loading?: "lazy" | "eager";  // 新增loading属性
}

const { loading = "eager", ...otherProps } = Astro.props;
```

```astro
{isLocal && img && <Image src={img} loading={loading} .../>}
{!isLocal && <img src={src} loading={loading} .../>}
```

**好处**：
- 统一的loading控制接口
- 同时支持Astro Image组件和原生img标签

### 4. FullscreenWallpaper组件优化
**位置**: `src/components/misc/FullscreenWallpaper.astro`

应用相同的优化策略：
- 随机模式只渲染一张
- 轮播模式第一张eager，其他lazy

### 5. 简化客户端逻辑
**位置**: `src/layouts/Layout.astro`

**删除的代码**：
```javascript
// 旧代码：客户端随机选择（已删除）
if (validItems.length > 1 && !siteConfig.banner.carousel?.enable) {
    const randomIndex = Math.floor(Math.random() * validItems.length);
    // ... 隐藏所有图片，只显示随机的一张
}
```

**好处**：
- 随机选择移到服务端，HTML更小
- 减少客户端JavaScript执行
- 避免布局抖动（CLS优化）

## 性能提升

### 随机模式（carousel.enable: false）
- ✅ 图片请求：从 6张 → 1张（减少83%）
- ✅ 传输大小：从 ~8MB → ~1.3MB（假设每张图1.3MB）
- ✅ 首屏加载时间：显著减少

### 轮播模式（carousel.enable: true）
- ✅ 关键路径优化：只有第一张图片阻塞渲染
- ✅ 渐进式加载：后续图片在滚动或空闲时加载
- ✅ 带宽优化：按需加载，不浪费用户流量
- ✅ LCP改善：第一张图片优先加载

### 全屏壁纸
- ✅ 同样的优化适用于fullscreen wallpaper模式

## 测试验证

### 随机模式测试
```bash
# 设置 carousel.enable: false
pnpm build
grep 'Mobile banner' dist/index.html | wc -l   # 输出：1（原本6）
grep 'Desktop banner' dist/index.html | wc -l  # 输出：1（原本6）
```

### 轮播模式测试
```bash
# 设置 carousel.enable: true
pnpm build
grep -o 'loading="eager"' dist/index.html      # 第一张图片
grep -o 'loading="lazy"' dist/index.html       # 其他5张图片
```

## 兼容性

- ✅ 保持所有原有功能
- ✅ 视觉效果无变化
- ✅ 轮播交互无变化
- ✅ 移动端/桌面端分别处理
- ✅ 浏览器原生lazy loading（现代浏览器广泛支持）

## 文件变更清单

1. `src/layouts/MainGridLayout.astro` - 主要优化逻辑
2. `src/components/misc/ImageWrapper.astro` - 添加loading属性支持
3. `src/components/misc/FullscreenWallpaper.astro` - 应用相同优化
4. `src/layouts/Layout.astro` - 简化客户端轮播逻辑

## 后续建议

1. **可选：预加载下一张**
   - 在轮播模式下，可以添加 `<link rel="prefetch">` 预加载下一张图片
   
2. **可选：响应式图片**
   - 使用 `srcset` 为不同屏幕尺寸提供不同大小的图片

3. **监控**
   - 使用 Lighthouse/Web Vitals 监控 LCP、CLS 等指标
   - 验证实际性能提升

## 总结

此次优化通过以下三个核心改进显著提升了页面性能：

1. **服务端随机选择** - 随机模式下减少83%的图片加载
2. **原生Lazy Loading** - 轮播模式下渐进式加载图片
3. **优先级优化** - 第一张图片立即加载，优化LCP

实现了在不影响用户体验的前提下，大幅减少不必要的资源加载，特别是在移动网络环境下效果显著。
