# AudioClaude

Voice interface to pilot [Claude Code](https://claude.ai/claude-code) remotely from iPhone via AirPods — powered by Kyutai STT/TTS and Mimi codec.

Dictate instructions from your iPhone (commuting, walking, shopping), hear Claude's responses in natural voice through your AirPods, and find your session intact when you get back to your Mac.

## How it works

```
iPhone (mic/speaker)  <──SSH tunnel──>  Mac (Claude Code in tmux)
        │                                       │
    Mimi codec                           Kyutai STT 1B
   (1.1 kbps)                           Kyutai TTS 1.6B
        │                                Mimi codec
        └──────── WebSocket (port 9000) ────────┘
```

## Stack

- **Server**: Python 3.12+, asyncio, websockets
- **AI Audio**: Kyutai STT 1B (en/fr), Kyutai TTS 1.6B, Mimi codec (rustymimi)
- **iOS Client**: Swift, SwiftUI, AVFoundation, iOS 17+
- **Infra**: tmux, SSH tunnel, WebSocket

## Getting started

```bash
# Install dependencies
pip install -e "server[dev]"

# Start the server
python -m audioclaude.server

# Run tests
pytest server/tests/

# Lint & format
ruff check server/
ruff format server/
```

## Project status

Early development — see [plans/](plans/) for the full architecture and roadmap.

## Acknowledgments

This project uses speech-to-text, text-to-speech models and the Mimi audio codec from [Kyutai](https://github.com/kyutai-labs/moshi), distributed under MIT/Apache 2.0 (code) and CC-BY 4.0 (model weights).

## License

[MIT](LICENSE)
