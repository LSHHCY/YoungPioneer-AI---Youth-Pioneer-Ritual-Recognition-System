# YoungPioneerAI 插件使用文档

## 插件概述

### 基本信息
- **插件ID**: 8yzjxf6iofsw
- **插件名称**: 多模态生成接口
- **用途**: 少先队礼仪识别 - 系红领巾步骤分析
- **模型**: 阿里云 DashScope qwen-vl-plus
- **API Endpoint**: https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation

### 核心功能
- 视频内容分析
- 图片序列分析
- 步骤级别的详细评价
- 结构化JSON输出
- 友好的改进建议

## API 调用方式

### 请求格式

#### 视频模式
```typescript
const response = await fetch(
  'https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'qwen-vl-plus',
      input: {
        messages: [
          {
            role: 'user',
            content: [
              {
                video: 'https://your-bucket.supabase.co/storage/v1/object/public/scarf-videos/demo.mp4'
              },
              {
                text: '请分析学生系红领巾的动作...'
              }
            ]
          }
        ]
      }
    })
  }
);
```

#### 图片序列模式
```typescript
const response = await fetch(
  'https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'qwen-vl-plus',
      input: {
        messages: [
          {
            role: 'user',
            content: [
              { image: 'https://example.com/step1.jpg' },
              { image: 'https://example.com/step2.jpg' },
              { image: 'https://example.com/step3.jpg' },
              { text: '这是一组学生系红领巾的关键帧...' }
            ]
          }
        ]
      }
    })
  }
);
```

### 响应格式

#### 成功响应
```json
{
  "request_id": "d824d5ea-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "output": {
    "choices": [
      {
        "finish_reason": "stop",
        "message": {
          "role": "assistant",
          "content": "{\"status\": \"success\", \"analysis\": {\"steps\": [...], \"overall_comment\": \"...\"}}"
        }
      }
    ]
  },
  "usage": {
    "output_tokens": 256,
    "input_tokens": 1500
  }
}
```

#### 分析结果JSON结构
```json
{
  "status": "success",
  "analysis": {
    "steps": [
      {
        "step_index": 1,
        "step_name": "披肩交叉",
        "status": "pass",
        "comment": "动作标准,右角正确放在左角下面"
      },
      {
        "step_index": 2,
        "step_name": "拉角到右",
        "status": "pass",
        "comment": "右角顺利拉到右边,左角保持不动"
      },
      {
        "step_index": 3,
        "step_name": "绕圈穿出",
        "status": "fail",
        "comment": "右角没有正确穿过空隙,建议重新练习这个步骤"
      },
      {
        "step_index": 4,
        "step_name": "抽紧调整",
        "status": "pass",
        "comment": "最后抽紧动作完成"
      }
    ],
    "overall_comment": "整体动作基本规范,第三步需要加强练习。建议多看示范视频,注意右角穿过空隙的角度。"
  }
}
```

## 提示词设计

### 系红领巾步骤分析提示词
```
请仔细观看这段视频,视频内容是一名学生正在系红领巾。请按照以下四个标准步骤的时间顺序,检查该学生的动作流程是否正确:

1. 【第一步】将红领巾披在肩上,右角放在左角下面,两角交叉。
2. 【第二步】将右角经过左角前面,拉到右边,左角不动。
3. 【第三步】右角经左右两角交叉的空隙中拉出,绕过左角一圈。
4. 【第四步】将右角从此圈中拉出,抽紧。

请以JSON格式输出分析结果,格式如下:
{
  "status": "success",
  "analysis": {
    "steps": [
      {
        "step_index": 1,
        "step_name": "披肩交叉",
        "status": "pass",
        "comment": "动作标准,右角正确放在左角下面"
      },
      ...
    ],
    "overall_comment": "整体动作流畅规范,四个步骤都完成得很好!继续保持!"
  }
}

如果某个步骤有问题,status 设为 "fail",并在 comment 中给出具体的改进建议。
```

## Edge Function 实现

### 完整代码示例
```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const { videoUrl, type } = await req.json();
    
    if (type === 'scarf') {
      const apiKey = Deno.env.get('YOUNGPIONEER_API_KEY') || '[YOUR_API_KEY]';
      
      const response = await fetch(
        'https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation',
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${apiKey}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            model: 'qwen-vl-plus',
            input: {
              messages: [
                {
                  role: 'user',
                  content: [
                    { video: videoUrl },
                    { text: '分析提示词...' }
                  ]
                }
              ]
            }
          }),
        }
      );

      const result = await response.json();
      const content = result.output?.choices?.[0]?.message?.content || '';
      
      // 解析JSON结果
      const analysisResult = JSON.parse(content);
      
      // 构建友好的描述文本
      let description = '## 步骤分析\n\n';
      analysisResult.analysis.steps.forEach((step: any) => {
        const statusIcon = step.status === 'pass' ? '✅' : '❌';
        description += `${statusIcon} **步骤${step.step_index}: ${step.step_name}**\n`;
        description += `   ${step.comment}\n\n`;
      });
      
      description += `## 总体评价\n\n${analysisResult.analysis.overall_comment}`;

      return new Response(
        JSON.stringify({
          success: true,
          description: description,
          rawAnalysis: analysisResult,
        }),
        { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }
  } catch (error) {
    return new Response(
      JSON.stringify({
        success: false,
        error: error.message,
      }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

## 前端集成

### API 调用
```typescript
// src/db/api.ts
export const recognitionApi = {
  recognizeVideoByUrl: async (data: { videoUrl: string; type: string }) => {
    const { data: result, error } = await supabase.functions.invoke('video-recognition', {
      body: data,
    });

    if (error) throw error;
    return result;
  },
};
```

### 使用示例
```typescript
// 在组件中调用
const recognizeVideo = async (videoUrl: string) => {
  try {
    const response = await recognitionApi.recognizeVideoByUrl({
      videoUrl: videoUrl,
      type: 'scarf',
    });

    if (response.success) {
      // 显示友好的描述文本
      console.log(response.description);
      
      // 使用原始分析结果
      const analysis = response.rawAnalysis;
      analysis.analysis.steps.forEach(step => {
        console.log(`步骤${step.step_index}: ${step.status}`);
      });
    }
  } catch (error) {
    console.error('识别失败:', error);
  }
};
```

## 错误处理

### 常见错误码

#### 401 Unauthorized
```json
{
  "code": "InvalidApiKey",
  "message": "Invalid API-key provided."
}
```
**解决方案**: 检查API Key是否正确配置

#### 400 Bad Request
```json
{
  "code": "InvalidParameter",
  "message": "The video URL is invalid or cannot be downloaded."
}
```
**解决方案**: 
- 确认视频URL可公开访问
- 检查视频格式是否支持
- 验证URL格式正确

#### 429 Too Many Requests
```json
{
  "code": "Throttling.RateQuota",
  "message": "Request was denied due to request throttling."
}
```
**解决方案**: 
- 降低请求频率
- 实现请求队列
- 升级API套餐

#### 500 Internal Error
```json
{
  "code": "InternalError",
  "message": "An internal error occurred."
}
```
**解决方案**: 
- 重试请求
- 检查视频内容是否合规
- 联系技术支持

## 最佳实践

### 1. 视频要求
- **时长**: 建议5秒以内
- **格式**: MP4, MOV
- **大小**: 不超过50MB
- **分辨率**: 720p或以上
- **帧率**: 24fps或以上

### 2. 提示词优化
- 明确指定分析步骤
- 要求JSON格式输出
- 提供输出示例
- 包含改进建议要求

### 3. 错误处理
- 实现完整的try-catch
- 记录详细的错误日志
- 提供友好的错误提示
- 实现降级策略

### 4. 性能优化
- 添加超时控制(60秒)
- 实现请求缓存
- 使用异步处理
- 显示处理进度

### 5. 用户体验
- 显示等待提示
- 格式化显示结果
- 使用图标标识状态
- 提供重试功能

## 测试用例

### 测试1: 标准动作视频
```typescript
const videoUrl = 'https://example.com/standard_scarf_tying.mp4';
const result = await recognizeVideo(videoUrl);

// 预期结果: 所有步骤status为"pass"
expect(result.rawAnalysis.analysis.steps.every(s => s.status === 'pass')).toBe(true);
```

### 测试2: 错误动作视频
```typescript
const videoUrl = 'https://example.com/wrong_scarf_tying.mp4';
const result = await recognizeVideo(videoUrl);

// 预期结果: 至少一个步骤status为"fail"
expect(result.rawAnalysis.analysis.steps.some(s => s.status === 'fail')).toBe(true);
```

### 测试3: 无效视频URL
```typescript
const videoUrl = 'https://invalid-url.com/video.mp4';

// 预期结果: 抛出错误
await expect(recognizeVideo(videoUrl)).rejects.toThrow();
```

## 监控和日志

### 关键日志点
```typescript
console.log('收到视频识别请求:', { videoUrl, type });
console.log('开始调用 YoungPioneerAI 多模态生成接口...');
console.log('YoungPioneerAI API响应状态:', response.status);
console.log('YoungPioneerAI API响应:', JSON.stringify(result));
console.log('解析后的分析结果:', analysisResult);
```

### 性能指标
- API调用时间: 20-60秒
- 成功率: >95%
- 错误率: <5%
- 超时率: <2%

## 相关资源

- **插件ID**: 8yzjxf6iofsw
- **API文档**: https://dashscope.aliyuncs.com/
- **模型文档**: https://help.aliyun.com/zh/dashscope/
- **Edge Function**: supabase/functions/video-recognition/index.ts
- **前端API**: src/db/api.ts

## 版本历史

- 2026-01-16: 初始版本,替换原有通义千问compatible-mode接口
- 2026-01-16: 添加结构化JSON输出支持
- 2026-01-16: 实现步骤级别的详细分析
- 2026-01-16: 添加友好的格式化显示
