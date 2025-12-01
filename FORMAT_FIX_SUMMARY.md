# 格式修复总结

## 问题
之前的修改将 `ImageWrapper.astro` 文件的缩进从 tab 改成了空格，这不符合项目的代码风格。

## 解决方案
已将 `ImageWrapper.astro` 文件恢复为使用 tab 缩进，同时保留了功能性的修改（添加 `loading` 属性支持）。

## 验证
使用 `cat -A` 命令确认文件现在使用 tab (`^I`) 而不是空格进行缩进。

## 功能性更改（保留）
1. 添加 `loading?: "lazy" | "eager"` 到 Props 接口
2. 在组件解构中添加 `loading = "eager"` 默认值
3. 在 Image 和 img 标签中添加 `loading={loading}` 属性

## 格式更改（已修复）
- ✅ 恢复使用 tab 缩进（而不是空格）
- ✅ 保持原有代码风格
- ✅ 只修改必要的功能代码

## 构建验证
✅ `pnpm build` 成功完成，没有错误
