# 通义千问qwen-vl-plus API快速参考

## 关键要点

### ⚠️ 最重要的发现
**通义千问的多模态消息格式与OpenAI完全不同!**
- ❌ 不要使用`type`字段
- ❌ 不要使用`image_url`对象
- ✅ 直接使用`image`或`text`作为对象的唯一键

## 正确格式

### 多模态请求(图片+文本)
```typescript
const response = await fetch(
  'https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'qwen-vl-plus',
      messages: [
        {
          role: 'user',
          content: [
            {
              image: 'data:image/jpeg;base64,/9j/4AAQ...'
            },
            {
              text: '请分析这张图片'
            }
          ]
        }
      ]
    })
  }
);
```

### 纯文本请求
```typescript
const response = await fetch(
  'https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'qwen-vl-plus',
      messages: [
        {
          role: 'system',
          content: '你是一位助手'
        },
        {
          role: 'user',
          content: '你好'
        }
      ]
    })
  }
);
```

## 错误格式对比

### ❌ 错误1: OpenAI风格
```typescript
{
  role: 'user',
  content: [
    {
      type: 'image_url',  // ❌ 错误
      image_url: {        // ❌ 错误
        url: 'data:image/jpeg;base64,...'
      }
    },
    {
      type: 'text',       // ❌ 错误
      text: '请分析'
    }
  ]
}
```

### ❌ 错误2: 包含type字段
```typescript
{
  role: 'user',
  content: [
    {
      type: 'image',      // ❌ 多余的type字段
      image: 'data:image/jpeg;base64,...'
    },
    {
      type: 'text',       // ❌ 多余的type字段
      text: '请分析'
    }
  ]
}
```

### ✅ 正确: 通义千问格式
```typescript
{
  role: 'user',
  content: [
    {
      image: 'data:image/jpeg;base64,...'  // ✅ 只有image字段
    },
    {
      text: '请分析'  // ✅ 只有text字段
    }
  ]
}
```

## 响应格式

### 成功响应
```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "这是AI的回复内容"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 50,
    "total_tokens": 150
  }
}
```

### 错误响应
```json
{
  "error": {
    "message": "错误描述",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```

## 常见问题

### Q1: 为什么我的请求一直失败?
**A**: 检查是否使用了`type`字段或`image_url`对象,这些都是错误的格式。

### Q2: 图片应该如何编码?
**A**: 使用base64编码,格式为`data:image/jpeg;base64,{base64数据}`

### Q3: 可以同时发送多张图片吗?
**A**: 可以,在content数组中添加多个image对象即可。

### Q4: 文本和图片的顺序重要吗?
**A**: 通常先放图片,后放文本提示,但顺序不是强制的。

### Q5: 支持哪些图片格式?
**A**: 支持JPEG、PNG等常见格式,建议使用JPEG以减小大小。

## 最佳实践

### 1. 图片大小控制
```typescript
// 压缩图片到合适大小
const compressImage = async (imageData: string): Promise<string> => {
  const img = new Image();
  img.src = imageData;
  await new Promise(resolve => img.onload = resolve);
  
  const canvas = document.createElement('canvas');
  const maxWidth = 1920;
  const maxHeight = 1080;
  let width = img.width;
  let height = img.height;
  
  if (width > maxWidth || height > maxHeight) {
    const ratio = Math.min(maxWidth / width, maxHeight / height);
    width *= ratio;
    height *= ratio;
  }
  
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  ctx?.drawImage(img, 0, 0, width, height);
  
  return canvas.toDataURL('image/jpeg', 0.8);
};
```

### 2. 错误处理
```typescript
try {
  const response = await fetch(API_URL, options);
  
  if (!response.ok) {
    const errorText = await response.text();
    console.error('API错误:', errorText);
    throw new Error(`API调用失败: ${response.status}`);
  }
  
  const data = await response.json();
  return data.choices[0].message.content;
} catch (error) {
  console.error('请求失败:', error);
  // 实现降级策略
  return '默认回复';
}
```

### 3. 日志记录
```typescript
console.log('开始调用API...');
console.log('图片大小:', base64Data.length);
console.log('API响应状态:', response.status);
console.log('API响应结果:', JSON.stringify(data));
```

## 调试技巧

### 1. 验证API Key
```bash
curl -X POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-vl-plus","messages":[{"role":"user","content":"你好"}]}'
```

### 2. 测试图片格式
先测试纯文本请求,确认API Key有效后,再测试图片请求。

### 3. 检查请求体大小
确保请求体不超过API限制(通常为10MB)。

## 相关资源

- 通义千问官方文档: https://help.aliyun.com/zh/dashscope/
- API参考: https://dashscope.aliyuncs.com/
- 模型列表: qwen-vl-plus, qwen-vl-max

## 版本历史

- 2026-01-16: 发现并修复type字段问题
- 2026-01-16: 创建快速参考文档
