# 附录 C · 多模态 Skill 完整代码示例

> 本附录包含第十二章（自定义 Skill 开发）中 12.9.2–12.9.6 节多模态 Skill 的完整代码实现。正文中仅保留核心逻辑与架构说明，完整版本请参阅本节。

---

## C.1 PPT 生成 Skill 完整代码

```python
# src/ppt_generator.py
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RgbColor
from pptx.enum.text import PP_ALIGN

class PPTGenerator:
    """PPT 生成器"""
    
    def __init__(self, template_dir: str, style: str = "business"):
        self.prs = Presentation()
        self.style = self._load_style(style)
    
    def generate(self, content: dict, max_slides: int = 15) -> str:
        """生成 PPT 文件"""
        
        # 1. 创建标题页
        self._add_title_slide(
            title=content["title"],
            subtitle=content.get("subtitle", "")
        )
        
        # 2. 创建内容页
        for section in content["sections"][:max_slides - 1]:
            self._add_content_slide(section)
        
        # 3. 保存
        output_path = f"outputs/{content['filename']}.pptx"
        self.prs.save(output_path)
        return output_path
    
    def _add_title_slide(self, title: str, subtitle: str):
        """添加标题页"""
        slide_layout = self.prs.slide_layouts[0]  # Title Slide
        slide = self.prs.slides.add_slide(slide_layout)
        
        # 设置标题
        title_shape = slide.shapes.title
        title_shape.text = title
        title_para = title_shape.text_frame.paragraphs[0]
        title_para.font.size = Pt(44)
        title_para.font.bold = True
        title_para.font.color.rgb = self.style["primary_color"]
        
        # 设置副标题
        if subtitle:
            subtitle_shape = slide.placeholders[1]
            subtitle_shape.text = subtitle
    
    def _add_content_slide(self, section: dict):
        """添加内容页"""
        slide_layout = self.prs.slide_layouts[1]  # Title and Content
        slide = self.prs.slides.add_slide(slide_layout)
        
        # 标题
        title_shape = slide.shapes.title
        title_shape.text = section["heading"]
        
        # 内容要点
        body_shape = slide.placeholders[1]
        tf = body_shape.text_frame
        tf.clear()
        
        for i, point in enumerate(section["points"][:5]):
            p = tf.add_paragraph()
            p.text = f"• {point}"
            p.font.size = Pt(18)
            p.space_after = Pt(12)
            p.level = 0
```python

---

## C.2 播客生成 Skill 完整代码

```python
# src/podcast_generator.py
from TTS.api import TTS
from pydub import AudioSegment
import tempfile

class PodcastGenerator:
    """播客生成器"""
    
    def __init__(self, voice_model: str = "tts_models/en/vctk/vits"):
        self.tts = TTS(model_name=voice_model)
    
    async def generate(
        self,
        content: str,
        style: str = "single",
        duration_minutes: int = 15
    ) -> str:
        """生成播客音频"""
        
        # 1. 生成播客脚本
        script = self._generate_script(content, style, duration_minutes)
        
        # 2. 分段 TTS
        segments = []
        for segment in script["segments"]:
            audio_path = await self._synthesize_segment(segment)
            segments.append(audio_path)
        
        # 3. 合并音频 + 添加背景音乐
        final_audio = self._merge_segments(segments)
        
        # 4. 保存
        output_path = f"outputs/podcast_{script['title']}.mp3"
        final_audio.export(output_path, format="mp3", bitrate="128k")
        return output_path
    
    async def _synthesize_segment(self, segment: dict) -> str:
        """合成单个语音片段"""
        text = segment["text"]
        speaker = segment.get("speaker", "default")
        
        # 使用 TTS 生成
        with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
            self.tts.tts_to_file(
                text=text,
                speaker=speaker,
                file_path=f.name
            )
            return f.name
    
    def _merge_segments(self, segment_paths: list) -> AudioSegment:
        """合并音频片段"""
        combined = AudioSegment.empty()
        
        for path in segment_paths:
            segment = AudioSegment.from_wav(path)
            # 添加 0.5s 停顿
            combined += segment + AudioSegment.silent(duration=500)
        
        return combined
```

---

## C.3 图片生成 Skill 完整代码

```python
# src/image_generator.py
from openai import AsyncOpenAI
from PIL import Image
import aiohttp
import io

class ImageGenerator:
    """图片生成器"""
    
    def __init__(self, api_key: str, default_model: str = "dall-e-3"):
        self.client = AsyncOpenAI(api_key=api_key)
        self.default_model = default_model
    
    async def generate(
        self,
        prompt: str,
        style: str = "vivid",
        size: str = "1024x1024",
        n: int = 1
    ) -> list[str]:
        """生成图片"""
        
        # 1. 优化提示词
        optimized_prompt = self._enhance_prompt(prompt, style)
        
        # 2. 调用 API
        response = await self.client.images.generate(
            model=self.default_model,
            prompt=optimized_prompt,
            size=size,
            n=n,
            quality="standard" if size == "1024x1024" else "hd"
        )
        
        # 3. 下载并保存
        paths = []
        for i, img_data in enumerate(response.data):
            image_url = img_data.url
            
            async with aiohttp.ClientSession() as session:
                async with session.get(image_url) as resp:
                    image_bytes = await resp.read()
            
            # 保存
            output_path = f"outputs/generated_image_{i}.png"
            with open(output_path, "wb") as f:
                f.write(image_bytes)
            
            paths.append(output_path)
        
        return paths
    
    def _enhance_prompt(self, prompt: str, style: str) -> str:
        """增强提示词"""
        style_prefixes = {
            "realistic": "High quality, photorealistic, detailed: ",
            "illustration": "Digital illustration, artistic, vibrant colors: ",
            "3d": "3D render, octane render, cinematic lighting: ",
            "pixel": "Pixel art, retro game style, 16-bit: "
        }
        
        prefix = style_prefixes.get(style, "")
        return f"{prefix}{prompt}"
```python

---

## C.4 视频生成 Skill 完整代码

```python
# src/video_generator.py
from moviepy.editor import (
    ImageClip, TextClip, CompositeVideoClip,
    AudioFileClip, concatenate_videoclips
)
from moviepy.video.fx.all import fadein, fadeout

class VideoGenerator:
    """视频生成器"""
    
    def __init__(self, resolution: tuple = (1920, 1080), fps: int = 30):
        self.resolution = resolution
        self.fps = fps
    
    async def generate(
        self,
        scenes: list[dict],
        audio_path: str = None,
        subtitle_style: dict = None
    ) -> str:
        """生成视频"""
        
        clips = []
        
        for scene in scenes:
            # 创建画面
            if "image_path" in scene:
                clip = ImageClip(scene["image_path"])
            else:
                # 纯色背景 + 文字
                clip = self._create_text_scene(scene)
            
            # 设置时长
            clip = clip.set_duration(scene["duration"])
            
            # 添加转场
            clip = fadein(clip, 0.5)
            clip = fadeout(clip, 0.5)
            
            clips.append(clip)
        
        # 合并
        final_video = concatenate_videoclips(clips, method="compose")
        
        # 添加音频
        if audio_path:
            audio = AudioFileClip(audio_path)
            audio = audio.subclip(0, final_video.duration)
            final_video = final_video.set_audio(audio)
        
        # 保存
        output_path = "outputs/generated_video.mp4"
        final_video.write_videofile(
            output_path,
            fps=self.fps,
            codec="libx264",
            audio_codec="aac"
        )
        
        return output_path
    
    def _create_text_scene(self, scene: dict) -> ImageClip:
        """创建文字场景"""
        # 创建背景
        bg = ImageClip("templates/blank_bg.png")
        bg = bg.resize(self.resolution)
        
        # 添加文字
        text = TextClip(
            scene["text"],
            fontsize=60,
            color="white",
            font="Arial-Bold",
            size=self.resolution,
            method="caption"
        ).set_duration(scene["duration"])
        
        return CompositeVideoClip([bg, text])
```

---

## C.5 看板生成 Skill 完整代码

```python
# src/kanban_generator.py
from jinja2 import Template
import json

class KanbanGenerator:
    """看板生成器"""
    
    def __init__(self, template_dir: str = "templates"):
        self.template_dir = template_dir
    
    def generate_html(self, tasks: list[dict], columns: list[str] = None) -> str:
        """生成 HTML 看板"""
        
        if columns is None:
            columns = ["Backlog", "To Do", "In Progress", "Done"]
        
        # 按状态分组
        grouped = {col: [] for col in columns}
        for task in tasks:
            status = task.get("status", "Backlog")
            if status in grouped:
                grouped[status].append(task)
        
        # 渲染模板
        template_str = self._load_template("kanban.html")
        template = Template(template_str)
        
        html = template.render(
            columns=columns,
            tasks=grouped,
            title="Project Kanban"
        )
        
        # 保存
        output_path = "outputs/kanban.html"
        with open(output_path, "w", encoding="utf-8") as f:
            f.write(html)
        
        return output_path
    
    def generate_image(self, html_path: str) -> str:
        """将 HTML 看板转为图片"""
        from html2image import Html2Image
        
        hti = Html2Image(size=(1920, 1080))
        output_path = "outputs/kanban.png"
        
        hti.screenshot(
            html_file=html_path,
            save_as=output_path
        )
        
        return output_path
```
