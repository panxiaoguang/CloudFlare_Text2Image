<script setup>
import { ref } from 'vue'
import { Picture } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import OpenAI from 'openai'

// 初始化OpenAI客户端
const openai = new OpenAI({
  baseURL: `https://api.cloudflare.com/client/v4/accounts/${import.meta.env.VITE_CLOUDFLARE_ACCOUNTID}/ai/v1`, // 使用代理地址
  apiKey: import.meta.env.VITE_OPENAI_API_KEY, // 使用环境变量中的API密钥
  dangerouslyAllowBrowser: true, // 允许在浏览器环境中使用
  defaultHeaders: { 'Content-Type': 'application/json' }
})

// 定义响应式数据
const prompt = ref('')
const steps = ref(4)
const magicPrompt = ref(false)
const loading = ref(false)
const generatedImage = ref('')
const loadingProgress = ref(0)
const progressTimer = ref(null)
const finalPromptRef = ref('')
const dialogVisible = ref(false)

// 优化提示词的方法
const optimizePrompt = async (inputPrompt) => {
  try {
    const completion = await openai.chat.completions.create({
      model: '@cf/meta/llama-3.3-70b-instruct-fp8-fast',
      messages: [
        {
          role: 'system',
          content: `
          你是一个基于Flux.1模型的提示词生成机器人。根据用户的需求，自动生成符合Flux.1格式的绘画提示词。虽然你可以参考提供的模板来学习提示词结构和规律，但你必须具备灵活性来应对各种不同需求。最终输出应仅限提示词，无需任何其他解释或信息。你的回答必须全部使用英语进行回复我！

### **提示词生成逻辑**：

1. **需求解析**：从用户的描述中提取关键信息，包括：
   - 角色：外貌、动作、表情等。
   - 场景：环境、光线、天气等。
   - 风格：艺术风格、情感氛围、配色等。
   - 其他元素：特定物品、背景或特效。

2. **提示词结构规律**：
   - **简洁、精确且具象**：提示词需要简单、清晰地描述核心对象，并包含足够细节以引导生成出符合需求的图像。
   - **灵活多样**：参考下列模板和已有示例，但需根据具体需求生成多样化的提示词，避免固定化或过于依赖模板。
   - **符合Flux.1风格的描述**：提示词必须遵循Flux.1的要求，尽量包含艺术风格、视觉效果、情感氛围的描述，使用与Flux.1模型生成相符的关键词和描述模式。

3. **仅供你参考和学习的几种场景提示词**（你需要学习并灵活调整,"[ ]"中内容视用户问题而定）：
   - **角色表情集**：
场景说明：适合动画或漫画创作者为角色设计多样的表情。这些提示词可以生成展示同一角色在不同情绪下的表情集，涵盖快乐、悲伤、愤怒等多种情感。

提示词：An anime [SUBJECT], animated expression reference sheet, character design, reference sheet, turnaround, lofi style, soft colors, gentle natural linework, key art, range of emotions, happy sad mad scared nervous embarrassed confused neutral, hand drawn, award winning anime, fully clothed

[SUBJECT] character, animation expression reference sheet with several good animation expressions featuring the same character in each one, showing different faces from the same person in a grid pattern: happy sad mad scared nervous embarrassed confused neutral, super minimalist cartoon style flat muted kawaii pastel color palette, soft dreamy backgrounds, cute round character designs, minimalist facial features, retro-futuristic elements, kawaii style, space themes, gentle line work, slightly muted tones, simple geometric shapes, subtle gradients, oversized clothing on characters, whimsical, soft puffy art, pastels, watercolor

   - **全角度角色视图**：
场景说明：当需要从现有角色设计中生成不同角度的全身图时，如正面、侧面和背面，适用于角色设计细化或动画建模。

提示词：A character sheet of [SUBJECT] in different poses and angles, including front view, side view, and back view

   - **80 年代复古风格**：
场景说明：适合希望创造 80 年代复古风格照片效果的艺术家或设计师。这些提示词可以生成带有怀旧感的模糊宝丽来风格照片。

提示词：blurry polaroid of [a simple description of the scene], 1980s.

   - **智能手机内部展示**：
场景说明：适合需要展示智能手机等产品设计的科技博客作者或产品设计师。这些提示词帮助生成展示手机外观和屏幕内容的图像。

提示词：a iphone product image showing the iphone standing and inside the screen the image is shown

   - **双重曝光效果**：
场景说明：适合摄影师或视觉艺术家通过双重曝光技术创造深度和情感表达的艺术作品。

提示词：[Abstract style waterfalls, wildlife] inside the silhouette of a [man]’s head that is a double exposure photograph . Non-representational, colors and shapes, expression of feelings, imaginative, highly detailed

   - **高质感电影海报**：
场景说明：适合需要为电影创建引人注目海报的电影宣传或平面设计师。

提示词：A digital illustration of a movie poster titled [‘Sad Sax: Fury Toad’], [Mad Max] parody poster, featuring [a saxophone-playing toad in a post-apocalyptic desert, with a customized car made of musical instruments], in the background, [a wasteland with other musical vehicle chases], movie title in [a gritty, bold font, dusty and intense color palette].

   - **镜面自拍效果**：
场景说明：适合想要捕捉日常生活瞬间的摄影师或社交媒体用户。

提示词：Phone photo: A woman stands in front of a mirror, capturing a selfie. The image quality is grainy, with a slight blur softening the details. The lighting is dim, casting shadows that obscure her features. [The room is cluttered, with clothes strewn across the bed and an unmade blanket. Her expression is casual, full of concentration], while the old iPhone struggles to focus, giving the photo an authentic, unpolished feel. The mirror shows smudges and fingerprints, adding to the raw, everyday atmosphere of the scene.

   - **像素艺术创作**：
场景说明：适合像素艺术爱好者或复古游戏开发者创造或复刻经典像素风格图像。

提示词：[Anything you want] pixel art style, pixels, pixel art

   - **以上部分场景仅供你学习，一定要学会灵活变通，以适应任何绘画需求**：

4. **Flux.1提示词要点总结**：
   - **简洁精准的主体描述**：明确图像中核心对象的身份或场景。
   - **风格和情感氛围的具体描述**：确保提示词包含艺术风格、光线、配色、以及图像氛围等信息。
   - **动态与细节的补充**：提示词可包括场景中的动作、情绪、或光影效果等重要细节。
   - **其他更多规律请自己寻找**
---

**问答案例1**：
**用户输入**：一个80年代复古风格的照片。
**你的输出**：A blurry polaroid of a 1980s living room, with vintage furniture, soft pastel tones, and a nostalgic, grainy texture,  The sunlight filters through old curtains, casting long, warm shadows on the wooden floor, 1980s,

**问答案例2**：
**用户输入**：一个赛博朋克风格的夜晚城市背景
**你的输出**：A futuristic cityscape at night, in a cyberpunk style, with neon lights reflecting off wet streets, towering skyscrapers, and a glowing, high-tech atmosphere. Dark shadows contrast with vibrant neon signs, creating a dramatic, dystopian mood
          `
        },
        {
          role: 'user',
          content: inputPrompt
        }
      ]
    })

    if (!completion.choices || !completion.choices[0] || !completion.choices[0].message) {
      throw new Error('提示词优化返回数据格式不正确')
    }

    return completion.choices[0].message.content
  } catch (error) {
    console.error('提示词优化失败:', error)
    throw error
  }
}

// 生成图片的方法
const generateImage = async () => {
  if (!prompt.value.trim()) {
    ElMessage.warning('请输入图片描述')
    return
  }

  let finalPrompt = prompt.value
  if (magicPrompt.value) {
    try {
      loading.value = true
      finalPrompt = await optimizePrompt(prompt.value)
      finalPromptRef.value = finalPrompt
      ElMessage.success('提示词优化成功')
    } catch (error) {
      ElMessage.error('提示词优化失败，将使用原始提示词')
      finalPromptRef.value = prompt.value
    }
  } else {
    finalPromptRef.value = prompt.value
  }

  loading.value = true
  loadingProgress.value = 0
  
  // 启动虚拟进度条计时器
  if (progressTimer.value) {
    clearInterval(progressTimer.value)
  }
  progressTimer.value = setInterval(() => {
    if (loadingProgress.value < 99) {
      loadingProgress.value += 1
    }
  }, 100) // 每100ms增加1%，总共10秒

  try {
    const response = await fetch(`https://api.cloudflare.com/api/client/v4/accounts/${import.meta.env.VITE_CLOUDFLARE_ACCOUNTID}/ai/run/@cf/black-forest-labs/flux-1-schnell`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        prompt: finalPrompt,
        steps: steps.value
      })
    })

    if (!response.ok) {
      throw new Error('API请求失败')
    }

    const data = await response.json()
    console.log('API响应数据:', data)  // 添加日志以便调试
    if (!data || !data.result || !data.result.image) {
      throw new Error('API返回数据格式不正确')
    }
    generatedImage.value = `data:image/png;base64,${data.result.image}`
    ElMessage.success('图片生成成功')
  } catch (error) {
    console.error('生成图片失败:', error)
    ElMessage.error('生成图片失败，请重试')
  } finally {
    loading.value = false
    if (progressTimer.value) {
      clearInterval(progressTimer.value)
      progressTimer.value = null
    }
    loadingProgress.value = 0
  }
}
</script>

<template>
  <div class="container">
    <!-- 左侧控制面板 -->
    <div class="sidebar">
      <h2>AI 画图</h2>
      <el-form>
        <el-form-item label="Prompt">
          <el-input
            v-model="prompt"
            type="textarea"
            :rows="4"
            placeholder="请输入图片描述..."
          />
        </el-form-item>

        <el-form-item label="Steps">
          <el-slider
            v-model="steps"
            :min="1"
            :max="50"
            :step="1"
            show-input
          />
        </el-form-item>

        <el-form-item label="Magic Prompt">
          <el-switch v-model="magicPrompt" />
        </el-form-item>

        <el-form-item>
          <el-button type="primary" @click="generateImage" :loading="loading">
            <el-icon><Picture /></el-icon>
            Generate
          </el-button>
        </el-form-item>
      </el-form>
    </div>

    <!-- 右侧图片展示区 -->
    <div class="main-content">
      <div class="image-container">
        <template v-if="loading">
          <el-empty>
            <template #description>
              <el-space direction="vertical" alignment="center" :size="20">
                <el-text>图片生成中...</el-text>
                <el-progress type="circle" :percentage="loadingProgress" />
              </el-space>
            </template>
          </el-empty>
        </template>
        <template v-else-if="!generatedImage">
          <el-empty description="暂无图片" />
        </template>
        <template v-else>
          <img :src="generatedImage" alt="Generated Image" style="max-width: 100%; max-height: 100%; object-fit: contain; cursor: pointer;" @click="dialogVisible = true" />
          <el-dialog v-model="dialogVisible" title="Generated Image" width="90%" center :style="{ maxWidth: '1200px' }">
            <div style="display: flex; flex-direction: column; gap: 20px;">
              <img :src="generatedImage" alt="Generated Image" style="width: 100%; height: auto; max-height: 60vh; object-fit: contain;" />
              <div style="background-color: #f5f7fa; border-radius: 8px; padding: 20px; box-shadow: inset 0 2px 4px rgba(0,0,0,0.05);">
                <strong style="display: block; color: #000000; font-size: 14px; margin-bottom: 12px;">Prompt:</strong>
                <div style="color: #000000; line-height: 1.6; font-size: 14px; white-space: pre-wrap; word-break: break-word;">{{ finalPromptRef }}</div>
              </div>
            </div>
          </el-dialog>
        </template>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  height: 100vh;
  width: 100vw;
  margin: 0;
  padding: 0;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
  background: linear-gradient(135deg, #f5f7fa 0%, #ffffff 100%);
}

.sidebar {
  position: relative;
  width: 320px;
  padding: 24px;
  background-color: rgba(255, 255, 255, 0.95);
  border-right: 1px solid rgba(230, 230, 230, 0.5);
  overflow-y: auto;
  margin: 0;
  box-shadow: 4px 0 16px rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(10px);
}

.sidebar h2 {
  margin-bottom: 24px;
  color: #2c3e50;
  font-size: 24px;
  font-weight: 600;
  text-align: center;
}

.main-content {
  flex: 1;
  padding: 32px;
  background-color: transparent;
  overflow-y: auto;
  margin: 0;
}

.image-container {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 16px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.el-form-item {
  margin-bottom: 24px;
}

.el-input__wrapper,
.el-slider,
.el-switch {
  margin: 8px 0;
}

.el-button {
  width: 100%;
  height: 44px;
  font-size: 16px;
  font-weight: 500;
  border-radius: 8px;
  transition: all 0.3s ease;
  background: linear-gradient(135deg, #409EFF 0%, #3a8ee6 100%);
}

.el-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.3);
}

.el-dialog {
  border-radius: 16px;
  overflow: hidden;
}

.el-dialog__header {
  background-color: #f5f7fa;
  padding: 20px 24px;
}

.el-dialog__title {
  font-size: 20px;
  font-weight: 600;
  color: #2c3e50;
}

.el-dialog__body {
  padding: 24px;
}

.el-dialog__footer {
  padding: 16px 24px;
  background-color: #f5f7fa;
}

:deep(.el-input__wrapper),
:deep(.el-textarea__wrapper) {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  border-radius: 8px;
}

:deep(.el-slider__runway) {
  margin: 16px 0;
}

:deep(.el-progress-circle) {
  width: 120px !important;
  height: 120px !important;
}

.generated-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  cursor: pointer;
  border-radius: 12px;
  transition: transform 0.3s ease;
}

.generated-image:hover {
  transform: scale(1.02);
}
</style>
