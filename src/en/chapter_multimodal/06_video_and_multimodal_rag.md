# 22.6 Video Understanding and Multimodal RAG

> **Goal**: Master implementation patterns for video-understanding Agents and understand the architecture and engineering practice of multimodal RAG.

---

## Video Understanding: Images With a Time Axis

A video is a sequence of images with a timeline, but video understanding is not just frame-by-frame image analysis. A useful video Agent must understand **temporal causality**: what happened first, how an action evolved, how scenes changed, and how local events contribute to the overall story.

### Three Levels of Video Understanding

```python
VIDEO_UNDERSTANDING_LEVELS = {
    "Level 1: Frame-level understanding": {
        "capability": "Identify objects, text, and scenes in a single frame",
        "example": "A red car appears at 00:15",
        "technique": "Frame extraction + vision model",
        "difficulty": "⭐⭐",
    },
    "Level 2: Clip-level understanding": {
        "capability": "Understand actions and events across several seconds",
        "example": "The person stands up and walks toward the door",
        "technique": "Multi-frame reasoning or video-native model",
        "difficulty": "⭐⭐⭐",
    },
    "Level 3: Video-level understanding": {
        "capability": "Understand the topic, narrative, and causal chain of the whole video",
        "example": "This is a cooking tutorial for braised pork",
        "technique": "Long-video encoding + hierarchical summarization",
        "difficulty": "⭐⭐⭐⭐",
    },
}
```

---

## Path 1: Frame Extraction + Vision Model

This approach works with almost any multimodal model. The Agent samples key frames, sends them to a vision model, and asks the model to reason over the ordered frame sequence.

```python
from openai import OpenAI
import base64
import cv2

client = OpenAI()


class VideoUnderstandingAgent:
    """Video understanding Agent based on frame sampling."""

    def __init__(self, model: str = "gpt-4.1"):
        self.model = model

    def extract_key_frames(
        self,
        video_path: str,
        interval_seconds: float = 5.0,
        max_frames: int = 20,
    ) -> list[tuple[float, str]]:
        """Extract key frames from a video at a fixed interval.

        Returns:
            A list of `(timestamp_seconds, base64_image)` pairs.
        """
        cap = cv2.VideoCapture(video_path)
        fps = cap.get(cv2.CAP_PROP_FPS)
        total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        frame_interval = max(1, int(fps * interval_seconds))

        frames = []
        for frame_index in range(0, total_frames, frame_interval):
            if len(frames) >= max_frames:
                break

            cap.set(cv2.CAP_PROP_POS_FRAMES, frame_index)
            success, frame = cap.read()
            if not success:
                continue

            timestamp = frame_index / fps
            _, buffer = cv2.imencode(".jpg", frame)
            image_base64 = base64.b64encode(buffer).decode("utf-8")
            frames.append((timestamp, image_base64))

        cap.release()
        return frames

    def analyze(self, video_path: str, question: str) -> str:
        frames = self.extract_key_frames(video_path)
        content = [
            {
                "type": "text",
                "text": (
                    "Analyze the following video frames in chronological order. "
                    f"Question: {question}"
                ),
            }
        ]

        for timestamp, image_base64 in frames:
            content.append({"type": "text", "text": f"Frame at {timestamp:.1f}s"})
            content.append({
                "type": "image_url",
                "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"},
            })

        response = client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": content}],
        )
        return response.choices[0].message.content
```

This approach is simple and reliable, but it has limitations:

- It may miss short actions between sampled frames.
- Long videos require aggressive sampling or summarization.
- Audio, subtitles, and metadata need separate processing.

---

## Path 2: Hierarchical Video Summarization

For long videos, use a hierarchical strategy:

1. Split the video into clips.
2. Extract frames and transcripts for each clip.
3. Generate a local summary per clip.
4. Merge clip summaries into a global summary.
5. Answer user questions using the global summary and relevant clips.

```python
class HierarchicalVideoSummarizer:
    def __init__(self, frame_agent: VideoUnderstandingAgent):
        self.frame_agent = frame_agent

    def summarize_clips(self, clip_paths: list[str]) -> list[dict]:
        summaries = []
        for index, clip_path in enumerate(clip_paths):
            summary = self.frame_agent.analyze(
                clip_path,
                "Summarize the main actions, scene changes, visible text, and important objects.",
            )
            summaries.append({"clip_id": index, "path": clip_path, "summary": summary})
        return summaries

    def merge_summaries(self, clip_summaries: list[dict]) -> str:
        joined = "\n\n".join(
            f"Clip {item['clip_id']}: {item['summary']}" for item in clip_summaries
        )
        response = client.chat.completions.create(
            model="gpt-4.1",
            messages=[{
                "role": "user",
                "content": (
                    "Create a coherent whole-video summary from these clip summaries. "
                    "Preserve temporal order and causal relationships.\n\n" + joined
                ),
            }],
        )
        return response.choices[0].message.content
```

The important idea is to preserve temporal structure. A good video summary should not only say what appears, but also **when it appears and how events connect**.

---

## Multimodal RAG Architecture

Multimodal RAG extends text RAG by indexing and retrieving multiple modalities:

```text
Documents / Images / Audio / Video
        ↓
Parsing and Chunking
        ↓
Text chunks + image frames + audio transcripts + video clip summaries
        ↓
Embedding and Metadata Indexing
        ↓
Hybrid Retrieval
        ↓
Multimodal Context Assembly
        ↓
LLM / Multimodal LLM Answer Generation
```

A production multimodal RAG system usually stores several representations for each asset:

| Asset | Representation | Retrieval Use |
|------|----------------|---------------|
| Image | Caption, OCR text, visual embedding | Search by visual content or text |
| Audio | Transcript, speaker segments, timestamps | Search by spoken content |
| Video | Key frames, clip summaries, transcript | Search by event, object, or time |
| PDF | Text, layout blocks, page screenshots | Search by content and visual layout |

---

## A Minimal Multimodal Index

```python
from dataclasses import dataclass
from typing import Literal


@dataclass
class MultimodalChunk:
    id: str
    modality: Literal["text", "image", "audio", "video"]
    content: str
    source_path: str
    timestamp: float | None = None
    metadata: dict | None = None


class MultimodalIndex:
    def __init__(self):
        self.chunks: list[MultimodalChunk] = []

    def add_chunk(self, chunk: MultimodalChunk) -> None:
        self.chunks.append(chunk)

    def search(self, query: str, top_k: int = 5) -> list[MultimodalChunk]:
        """A minimal lexical baseline; use vector or hybrid retrieval in production."""
        scored = []
        query_terms = set(query.lower().split())
        for chunk in self.chunks:
            score = len(query_terms & set(chunk.content.lower().split()))
            if score > 0:
                scored.append((score, chunk))
        scored.sort(key=lambda item: item[0], reverse=True)
        return [chunk for _, chunk in scored[:top_k]]
```

In production, replace this baseline with:

- text embeddings for captions, transcripts, and summaries;
- image embeddings for visual similarity;
- BM25 for exact keyword matching;
- metadata filters for time ranges, source files, speakers, or document sections.

---

## Context Assembly for Multimodal Answers

The retrieval result must be assembled into a context that the model can actually use:

```python
def build_multimodal_context(chunks: list[MultimodalChunk]) -> str:
    lines = []
    for chunk in chunks:
        location = chunk.source_path
        if chunk.timestamp is not None:
            location += f" @ {chunk.timestamp:.1f}s"
        lines.append(
            f"[{chunk.modality.upper()}] {location}\n"
            f"{chunk.content}\n"
        )
    return "\n".join(lines)
```

Good context assembly should include:

- source path and timestamp;
- modality type;
- confidence or extraction method if available;
- enough surrounding context to interpret the chunk;
- references that can be shown to the user.

---

## Engineering Best Practices

- **Sample adaptively**: sample more densely around scene changes, subtitles, or detected actions.
- **Keep timestamps everywhere**: every frame, transcript segment, and summary should be traceable to time.
- **Separate raw evidence from generated summaries**: summaries are useful but can hallucinate.
- **Use hybrid retrieval**: combine keyword search, vector search, metadata filters, and reranking.
- **Control cost**: cache frame captions and clip summaries; avoid reprocessing the same media.
- **Design for auditability**: users should be able to jump from an answer to the exact frame or clip.

---

## Chapter Takeaways

- Video understanding requires temporal reasoning, not just image recognition.
- Frame extraction is the most universal implementation path; hierarchical summarization makes it scalable.
- Multimodal RAG indexes text, images, audio, and video through multiple representations.
- Timestamps and source references are essential for trustworthy answers.
- Production systems should combine raw evidence, generated summaries, and hybrid retrieval.

---

*Previous: [22.5 Computer Use and GUI Agents](./05_computer_use_agent.md)*
