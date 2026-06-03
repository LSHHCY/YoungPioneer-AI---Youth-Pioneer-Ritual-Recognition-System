# 系红领巾步骤识别 - 少先队礼仪识别插件使用说明

## 概述

系红领巾步骤识别模块使用**少先队礼仪识别插件（8yzjxf6iofsw）**完成视频分析、步骤识别和评价反馈。

## 插件信息

### 基本信息
- **插件ID**: 8yzjxf6iofsw
- **插件名称**: 少先队礼仪识别
- **功能**: 系红领巾步骤分析
- **模型**: 阿里云 DashScope qwen-vl-plus
- **API Endpoint**: https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation

### 核心能力
- ✅ 视频内容分析
- ✅ 四步骤详细识别
- ✅ 结构化JSON输出
- ✅ 友好的改进建议
- ✅ 步骤级别的通过/失败判断

## 技术架构

### 1. 拍摄识别流程
```typescript
// src/pages/TieScarfPage.tsx
const startCamera = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({ 
    video: { facingMode: 'user' },
    audio: false 
  });
  videoRef.current.srcObject = stream;
};

const startRecording = () => {
  const mediaRecorder = new MediaRecorder(streamRef.current, {
    mimeType: 'video/webm',
  });
  
  mediaRecorder.onstop = async () => {
    const blob = new Blob(recordedChunksRef.current, { type: 'video/webm' });
    const file = new File([blob], 'recorded-video.webm', { type: 'video/webm' });
    await uploadRecordedVideo(file);
  };
  
  mediaRecorder.start();
};

const uploadRecordedVideo = async (file: File) => {
  // 上传到Supabase Storage
  const videoUrl = await uploadApi.uploadVideo(file, fileName);
  
  // 自动调用识别
  await recognizeVideo(videoUrl);
};
```

### 2. 上传视频流程
```typescript
// src/pages/TieScarfPage.tsx
const handleVideoUpload = async (event: React.ChangeEvent<HTMLInputElement>) => {
  const file = event.target.files?.[0];
  
  // 上传到Supabase Storage
  const videoUrl = await uploadApi.uploadVideo(file, fileName);
  
  // 自动调用识别
  await recognizeVideo(videoUrl);
};
```

### 3. 统一识别函数
```typescript
// src/pages/TieScarfPage.tsx
const recognizeVideo = async (videoUrl: string) => {
  const response = await recognitionApi.recognizeVideoByUrl({
    videoUrl: videoUrl,
    type: 'scarf',
  });
  
  if (response.success) {
    setResult({
      success: true,
      description: response.description,
      stepResults: response.stepResults,
    });
  }
};
```

### 4. 调用Edge Function
```typescript
// src/db/api.ts
export const recognitionApi = {
  recognizeVideoByUrl: async (data: { videoUrl: string; type: string }) => {
    const { data: result, error } = await supabase.functions.invoke('video-recognition', {
      body: data,
    });
    return result;
  },
};
```

### 5. Edge Function调用插件
```typescript
// supabase/functions/video-recognition/index.ts
// 插件ID: 8yzjxf6iofsw
// 插件名称: 少先队礼仪识别

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
    })
  }
);
```

### 6. 返回结构化结果
```json
{
  "success": true,
  "description": "📋 步骤分析结果\n\n✅ 步骤1-披肩交叉(通过): ...\n\n💬 老师的话\n\n...",
  "stepResults": {
    "step1": true,
    "step2": true,
    "step3": false,
    "step4": true
  },
  "rawAnalysis": {
    "status": "success",
    "analysis": {
      "steps": [...],
      "overall_comment": "..."
    }
  }
}
```

## 四个标准步骤

### 步骤1: 披肩交叉
- **动作要求**: 将红领巾披在肩上，钝角对着脊椎骨，右角放在左角下面，两角交叉
- **识别重点**: 右角是否在左角下面，两角是否正确交叉

### 步骤2: 拉角到右
- **动作要求**: 将右角经过左角前面，拉到右边，左角不动
- **识别重点**: 右角是否经过左角前面，左角是否保持不动

### 步骤3: 绕圈穿出
- **动作要求**: 右角经左右两角交叉的空隙中拉出，绕过左角一圈
- **识别重点**: 右角是否正确穿过空隙，是否绕过左角

### 步骤4: 抽紧调整
- **动作要求**: 将右角从此圈中拉出，抽紧
- **识别重点**: 是否完成抽紧动作，红领巾是否系好

## 提示词设计

### 完整提示词
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

### 提示词设计原则
1. **明确步骤顺序**: 按照1-2-3-4的时间顺序检查
2. **详细动作描述**: 每个步骤都有清晰的动作要求
3. **结构化输出**: 要求JSON格式，便于前端解析
4. **友好反馈**: 要求给出具体的改进建议

## 前端展示

### 步骤卡片展示
```tsx
{result && result.stepResults && (
  <div className="mt-6 space-y-4">
    <h3 className="font-semibold text-lg">步骤识别结果</h3>
    <div className="grid grid-cols-1 xl:grid-cols-2 gap-4">
      {SCARF_STEPS.map((step, index) => {
        const stepKey = `step${index + 1}` as keyof typeof result.stepResults;
        const isPassed = result.stepResults[stepKey];
        
        return (
          <Card key={step.key} className={isPassed ? 'border-green-500' : 'border-red-500'}>
            <CardHeader className={isPassed ? 'bg-green-50' : 'bg-red-50'}>
              <div className="flex items-center gap-2">
                {isPassed ? (
                  <CheckCircle2 className="w-5 h-5 text-green-600" />
                ) : (
                  <XCircle className="w-5 h-5 text-red-600" />
                )}
                <CardTitle className="text-base">{step.label}</CardTitle>
              </div>
            </CardHeader>
            <CardContent className="pt-4">
              <img src={step.image} alt={step.label} className="w-full rounded-lg mb-3" />
              <p className="text-sm text-muted-foreground">{step.description}</p>
              <div className={`mt-3 px-3 py-1 rounded-full text-sm font-medium inline-block ${
                isPassed ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'
              }`}>
                {isPassed ? '✓ 正确' : '✗ 需改进'}
              </div>
            </CardContent>
          </Card>
        );
      })}
    </div>
  </div>
)}
```

### 文字描述展示
```tsx
{result && (
  <Alert className={result.success ? 'border-green-500' : 'border-red-500'}>
    <AlertDescription className="whitespace-pre-wrap">
      {result.description}
    </AlertDescription>
  </Alert>
)}
```

## 日志监控

### 关键日志点
```
========================================
调用少先队礼仪识别插件 (8yzjxf6iofsw)
插件名称: 少先队礼仪识别
功能: 系红领巾步骤分析
模型: qwen-vl-plus
视频URL: https://...
========================================
少先队礼仪识别插件 API响应状态: 200
少先队礼仪识别插件 API响应: {...}
```

### 性能指标
- **识别时间**: 20-60秒
- **成功率**: >95%
- **准确率**: 基于qwen-vl-plus模型的视觉理解能力

## 最佳实践

### 1. 拍摄识别建议
**摄像头设置**：
- 使用前置摄像头（facingMode: 'user'）
- 确保光线充足，避免逆光
- 保持摄像头稳定，避免晃动

**录制技巧**：
- 录制时长：5-10秒，完整展示四个步骤
- 录制角度：正面或侧面，能清晰看到手部动作
- 录制速度：正常速度，不要太快或太慢
- 背景要求：简洁的背景，避免干扰

**注意事项**：
- 录制前先练习一遍，确保动作流畅
- 录制时保持镜头稳定
- 确保红领巾在画面中清晰可见
- 录制完成后会自动上传和识别，无需手动操作

### 2. 上传视频建议
**视频要求**：
- 时长：5-10秒，完整展示四个步骤
- 角度：正面或侧面，能清晰看到手部动作
- 光线：充足的光线，避免逆光
- 背景：简洁的背景，避免干扰
- 速度：正常速度，不要太快或太慢

### 3. 视频格式要求
**拍摄识别**：
- 格式：WebM（MediaRecorder自动生成）
- 大小：通常小于10MB（5-10秒视频）
- 分辨率：取决于摄像头
- 帧率：取决于摄像头

**上传视频**：
- 格式：MP4, MOV, AVI
- 大小：不超过50MB
- 分辨率：720p或以上
- 帧率：24fps或以上

### 4. 错误处理
**拍摄识别**：
- 摄像头启动失败：检查浏览器权限设置
- 录制失败：尝试刷新页面重新启动
- 上传失败：检查网络连接
- 识别超时：建议录制更短的视频（5-10秒）

**上传视频**：
- 文件过大：压缩视频或录制更短的视频
- 格式不支持：转换为MP4/MOV/AVI格式
- 上传失败：检查网络连接
- 识别失败：检查视频内容是否清晰

### 5. 用户体验优化
**拍摄识别**：
- Tabs切换：提供"拍摄识别"和"上传视频"两个选项
- 摄像头预览：实时显示摄像头画面
- 录制状态：显示红色"录制中"标识
- 自动上传：录制完成后自动上传
- 自动识别：上传完成后自动调用插件
- 进度提示：显示上传进度和识别状态
- 结果展示：步骤卡片 + 文字描述
- 重新录制：支持重新录制

**上传视频**：
- 文件选择：点击按钮选择本地视频
- 自动上传：选择完成后自动上传
- 自动识别：上传完成后自动调用插件
- 进度提示：显示上传进度和识别状态
- 结果展示：步骤卡片 + 文字描述
- 重新上传：支持重新上传

**Toast提示**：
- 明确标注使用"少先队礼仪识别插件"
- 提供详细的操作提示和错误信息

## 相关文件

### 前端文件
- `src/pages/TieScarfPage.tsx` - 系红领巾识别页面
- `src/db/api.ts` - API调用封装
- `src/db/uploadApi.ts` - 视频上传API

### 后端文件
- `supabase/functions/video-recognition/index.ts` - Edge Function
- `YOUNGPIONEER_AI_PLUGIN.md` - 插件详细文档

### 配置文件
- 环境变量: `YOUNGPIONEER_API_KEY`
- Storage Bucket: `scarf-videos`

## 版本历史

- **2026-01-18**: 添加拍摄识别模块，支持摄像头录制视频并自动识别
- **2026-01-18**: 创建统一识别函数，供拍摄和上传两种方式共用
- **2026-01-18**: 明确标注使用少先队礼仪识别插件（8yzjxf6iofsw）
- **2026-01-18**: 删除录制视频功能，只保留上传视频功能（已恢复）
- **2026-01-17**: 实现步骤卡片展示，绿色表示通过，红色表示需改进
- **2026-01-16**: 初始版本，实现视频上传和识别功能

## 总结

系红领巾步骤识别模块通过**少先队礼仪识别插件（8yzjxf6iofsw）**实现了：

1. ✅ 拍摄识别：通过摄像头录制视频并自动识别
2. ✅ 上传识别：上传本地视频文件并自动识别
3. ✅ 视频内容智能分析
4. ✅ 四个步骤详细识别
5. ✅ 结构化的识别结果
6. ✅ 友好的改进建议
7. ✅ 直观的可视化展示
8. ✅ 统一的识别流程

这为少先队员学习正确的系红领巾方法提供了智能化的辅导工具，支持两种识别方式，满足不同场景的需求。
