# dev-claude-reel

A Claude Code skill for making a short vertical (9:16, ~50 s) cinematic 3D reel of a conversation between a developer and a "Claude" character.

It covers the whole pipeline: script, ElevenLabs voices, a three.js scene rendered frame by frame in headless Chrome, shots and editing, the mix, and the final mp4. It also includes the QA rules that stop the result from looking buggy (floating characters, jitter, z-fighting, subtitles over faces).

Characters, set and story are left to you (or to Claude).

## Install

Copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/<your-username>/dev-claude-reel.git
cp -r dev-claude-reel/dev-claude-reel ~/.claude/skills/
```

Then ask Claude Code something like: "Make a dev-Claude reel about late-night debugging."

## Requirements

- Node 22+
- Chrome or Edge
- ffmpeg and ffprobe on PATH
- An ElevenLabs API key in `ELEVENLABS_API_KEY` (the free plan works for testing)
- A GPU (the skill refuses to render on CPU-only WebGL)

## License

MIT
