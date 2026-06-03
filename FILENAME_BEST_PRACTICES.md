# Supabase Storage 文件名最佳实践

## 问题背景

### 错误现象
```
{
  "statusCode": "400",
  "error": "InvalidKey",
  "message": "Invalid key: scarf-videos/1768558218640-屏幕录制 2026-01-16 180855.mp4"
}
```

### 根本原因
Supabase Storage对文件名有严格的要求:
- ❌ 不支持空格
- ❌ 不支持中文字符
- ❌ 不支持某些特殊字符

## 解决方案

### 文件名清理函数
```typescript
const cleanFileName = file.name
  .replace(/[\s\u4e00-\u9fa5]/g, '-')  // 将空格和中文替换为连字符
  .replace(/[^a-zA-Z0-9.-]/g, '')      // 移除其他特殊字符
  .replace(/-+/g, '-')                 // 将多个连字符合并为一个
  .toLowerCase();                      // 转为小写
```

### 清理规则说明

#### 1. 空格和中文字符处理
```typescript
.replace(/[\s\u4e00-\u9fa5]/g, '-')
```
- `\s`: 匹配所有空白字符(空格、制表符等)
- `\u4e00-\u9fa5`: 匹配所有中文字符
- 替换为连字符`-`,保持文件名的可读性

**示例**:
- `屏幕录制 2026-01-16.mp4` → `----2026-01-16.mp4`

#### 2. 特殊字符移除
```typescript
.replace(/[^a-zA-Z0-9.-]/g, '')
```
- `[^...]`: 匹配不在括号内的字符
- `a-zA-Z0-9.-`: 允许的字符(字母、数字、点、连字符)
- 移除所有其他字符

**示例**:
- `video@#$%test.mp4` → `videotest.mp4`

#### 3. 连字符合并
```typescript
.replace(/-+/g, '-')
```
- `-+`: 匹配一个或多个连续的连字符
- 替换为单个连字符

**示例**:
- `----2026-01-16.mp4` → `-2026-01-16.mp4`

#### 4. 转换为小写
```typescript
.toLowerCase()
```
- 统一使用小写,避免大小写混淆

**示例**:
- `Video-Test.MP4` → `video-test.mp4`

## 完整示例

### 输入文件名
```
屏幕录制 2026-01-16 180855.mp4
```

### 清理过程
1. 替换空格和中文: `----2026-01-16-180855.mp4`
2. 移除特殊字符: `----2026-01-16-180855.mp4` (无变化)
3. 合并连字符: `-2026-01-16-180855.mp4`
4. 转为小写: `-2026-01-16-180855.mp4` (无变化)

### 最终文件名
```
scarf-videos/1768558218640--2026-01-16-180855.mp4
```

## 实际应用

### 录制视频上传
```typescript
const uploadRecordedVideo = async (file: File) => {
  const cleanFileName = file.name
    .replace(/[\s\u4e00-\u9fa5]/g, '-')
    .replace(/[^a-zA-Z0-9.-]/g, '')
    .replace(/-+/g, '-')
    .toLowerCase();
  
  const fileName = `scarf-videos/${Date.now()}-${cleanFileName}`;
  console.log('上传文件名:', fileName);
  
  const videoUrl = await uploadApi.uploadVideo(file, fileName);
  // ...
};
```

### 手动上传视频
```typescript
const handleVideoUpload = async (event: React.ChangeEvent<HTMLInputElement>) => {
  const file = event.target.files?.[0];
  if (!file) return;
  
  const cleanFileName = file.name
    .replace(/[\s\u4e00-\u9fa5]/g, '-')
    .replace(/[^a-zA-Z0-9.-]/g, '')
    .replace(/-+/g, '-')
    .toLowerCase();
  
  const fileName = `scarf-videos/${Date.now()}-${cleanFileName}`;
  console.log('上传文件名:', fileName);
  
  const videoUrl = await uploadApi.uploadVideo(file, fileName);
  // ...
};
```

## 其他注意事项

### 1. 文件扩展名保留
清理函数会保留文件扩展名中的点(`.`),确保文件类型正确识别。

### 2. 时间戳前缀
使用`Date.now()`作为前缀,确保文件名唯一性:
```typescript
const fileName = `scarf-videos/${Date.now()}-${cleanFileName}`;
```

### 3. 路径分隔符
使用正斜杠`/`作为路径分隔符,不要使用反斜杠`\`。

### 4. 文件名长度
虽然没有明确限制,但建议文件名(包括路径)不超过255个字符。

## 支持的字符集

### ✅ 允许的字符
- 小写字母: `a-z`
- 大写字母: `A-Z`
- 数字: `0-9`
- 点: `.`
- 连字符: `-`
- 下划线: `_` (虽然清理函数会移除,但Storage支持)

### ❌ 不允许的字符
- 空格: ` `
- 中文字符: `中文`
- 特殊符号: `@#$%^&*()`
- 括号: `()[]{}` 
- 引号: `"'`
- 斜杠: `\` (反斜杠)

## 测试用例

### 测试1: 中文文件名
```typescript
输入: "视频测试.mp4"
输出: "1768558218640-.mp4"
```

### 测试2: 空格文件名
```typescript
输入: "my video file.mp4"
输出: "1768558218640-my-video-file.mp4"
```

### 测试3: 混合字符
```typescript
输入: "屏幕录制 2026-01-16 180855.mp4"
输出: "1768558218640--2026-01-16-180855.mp4"
```

### 测试4: 特殊字符
```typescript
输入: "video@#$test(1).mp4"
输出: "1768558218640-videotest1.mp4"
```

### 测试5: 大写字母
```typescript
输入: "MyVideo.MP4"
输出: "1768558218640-myvideo.mp4"
```

## 调试技巧

### 1. 添加日志
```typescript
console.log('原始文件名:', file.name);
console.log('清理后文件名:', cleanFileName);
console.log('最终路径:', fileName);
```

### 2. 验证文件名
在上传前验证文件名是否符合规范:
```typescript
const isValidFileName = /^[a-zA-Z0-9.-]+$/.test(cleanFileName);
if (!isValidFileName) {
  console.error('文件名包含非法字符:', cleanFileName);
}
```

### 3. 错误处理
捕获并显示详细的错误信息:
```typescript
try {
  const videoUrl = await uploadApi.uploadVideo(file, fileName);
} catch (error) {
  console.error('上传错误:', error);
  if (error instanceof Error) {
    toast({
      title: '上传失败',
      description: error.message,
      variant: 'destructive',
    });
  }
}
```

## 相关资源

- Supabase Storage文档: https://supabase.com/docs/guides/storage
- 文件命名规范: https://supabase.com/docs/guides/storage/uploads/standard-uploads
- 正则表达式测试: https://regex101.com/

## 版本历史

- 2026-01-16: 发现并修复文件名包含空格和中文的问题
- 2026-01-16: 创建文件名清理最佳实践文档
