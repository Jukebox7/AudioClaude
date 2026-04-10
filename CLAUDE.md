# AudioClaude

Interface vocale distante iPhone -> Mac pour piloter Claude Code a distance. L'utilisateur dicte des directives depuis son iPhone (en mobilite, courses, trajet), entend les reponses de Claude en voix naturelle dans ses AirPods, et retrouve sa session intacte en rentrant chez lui.

## Contexte et vision

Ce projet permet de ne jamais "quitter" sa session de dev. Le Mac fait tourner Claude Code dans tmux, le serveur Python orchestre le pipeline audio (Kyutai STT pour transcrire la voix, Kyutai TTS pour lire les reponses, Mimi pour compresser l'audio a 1.1kbps), et l'iPhone sert de terminal vocal leger connecte via SSH.

L'utilisateur est expert web JS/TS mais debutant en Swift — privilegier les explications avec des analogies web quand on touche a l'iOS.

## Stack technique

- **Serveur** : Python 3.12+, asyncio, websockets
- **IA Audio** : Kyutai STT 1B (en_fr), Kyutai TTS 1.6B, Mimi codec (rustymimi)
- **Packages** : moshi>=0.2.11, rustymimi, sphn, sounddevice, websockets
- **Client iOS** : Swift, SwiftUI, AVFoundation, iOS 17+
- **Infra** : tmux, SSH tunnel, WebSocket (port 9000)
- **ML Framework** : PyTorch (CUDA/MPS) ou MLX (Apple Silicon)

## Commandes

- Install deps : `pip install -e "server[dev]"`
- Lancer serveur : `python -m audioclaude.server`
- Tests : `pytest server/tests/`
- Lint : `ruff check server/`
- Format : `ruff format server/`

## Structure

```
server/audioclaude/   # Serveur Python asyncio (Mac) — tout le travail lourd
ios/AudioClaude/      # App Swift (iPhone) — client leger micro+speaker
plans/                # Documentation, architecture, diagrammes Mermaid
scripts/              # Scripts utilitaires (setup, launch, tunnel SSH)
```

## Conventions

- Python : ruff, type hints, async/await pour tout l'IO, asyncio.Queue entre composants
- Swift : SwiftUI, async/await, @Observable
- Audio : toujours 24kHz mono PCM comme format intermediaire
- Mimi frames : toujours exactement frame_size=1920 samples (80ms)
- Protocole WS : binaire (1 byte type + payload) pour audio, JSON texte pour controle
- Pas de secrets dans le code — cles SSH dans ~/.ssh/
- Langue : francais pour la doc et les commentaires, anglais pour le code

## Points d'attention

- Mimi streaming exige des frames de taille fixe (1920 samples = 80ms a 24kHz)
- Sur GPU, les appels Mimi sont CUDAGraphes : meme taille d'input obligatoire
- Le STT 1B inclut un VAD semantique — l'utiliser pour detecter fin de parole
- Le TTS streaming commence a generer avant d'avoir tout le texte
- tmux capture-pane : ne capturer que le delta (diff avec capture precedente)
- Anti-echo : couper le micro pendant la lecture TTS (mode push-to-talk en phase 1)
- Les hooks Claude Code (PostToolUse, Stop, Notification) sont une alternative au polling tmux

## Auto-amelioration

Ce fichier CLAUDE.md doit rester a jour avec l'etat reel du projet. Quand tu travailles sur AudioClaude :

- **Si tu ajoutes un nouveau module ou package** : mets a jour la section Structure et Stack technique
- **Si tu decouvres un piege technique** : ajoute-le dans Points d'attention
- **Si une commande change** : mets a jour la section Commandes
- **Si une convention est decidee** : ajoute-la dans Conventions
- **Si un plan de phase est termine** : mets a jour plans/README.md pour refleter l'avancement
- **Si une decision d'architecture change** : mets a jour plans/00-architecture.md et ce fichier
- **Si tu apprends quelque chose sur les preferences de l'utilisateur** : sauvegarde en memoire (type feedback ou user)

Garder ce fichier sous 80 lignes. Si ca deborde, extraire dans les fichiers de plans/.

## References

- [Plans du projet](plans/README.md) — Architecture, diagrammes Mermaid, phases de dev
- [Kyutai Moshi](https://github.com/kyutai-labs/moshi) — Codec Mimi + infra
- [Kyutai DSM](https://github.com/kyutai-labs/delayed-streams-modeling) — STT + TTS
- [Dossier apprentissage Claude Code](/Users/paulbompard/Documents/workspace/learn/claude) — Bonnes pratiques hooks, settings, subagents
