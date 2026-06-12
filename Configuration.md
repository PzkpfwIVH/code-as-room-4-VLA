# 配置

## 组件需求：

conda(miniconda、anaconda均可)
blender 4.X版本（需要添加路径至环境变量）

## 配置过程

```bash
conda create -n code_as_room python=3.12 -y
conda activate code_as_room

pip install langchain-openai langchain-core openai pillow requests
```

写入.env文件：
```env
SCENEGEN_MODEL=gemini-3.1-pro-preview-thinking
SCENEGEN_BASE_URL=https://your-openai-compatible-endpoint/v1
SCENEGEN_API_KEY=your-api-key

SCENEGEN_VISION_MODEL=gpt-4o
SCENEGEN_VISION_BASE_URL=https://api.openai.com/v1
SCENEGEN_VISION_API_KEY=sk-your-vision-api-key

SCENEGEN_TEXTURE_MODEL=gemini-3-pro-image-preview
SCENEGEN_TEXTURE_BASE_URL=https://your-image-generation-endpoint
SCENEGEN_TEXTURE_API_KEY=your-texture-api-key
```
从上到下依次为LLM模型、VLM模型、能进行图片生成的VLM模型。