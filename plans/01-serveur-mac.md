# Phase 1 — Setup serveur Mac + dependances Kyutai

## Objectif

Installer et valider toutes les dependances sur le Mac, s'assurer que STT, TTS et Mimi fonctionnent individuellement.

---

## Etapes

### 1.1 — Environnement Python

```bash
# Creer un venv dedie
python3.12 -m venv .venv
source .venv/bin/activate

# Dependances Kyutai
pip install "moshi>=0.2.11"
pip install rustymimi
pip install sphn sounddevice julius

# Dependances serveur
pip install websockets asyncio-mqtt
```

**Verification** :
```bash
python -c "import moshi; print(moshi.__version__)"
python -c "import rustymimi; print('rustymimi OK')"
```

### 1.2 — Telecharger les modeles

```bash
# STT 1B (francais + anglais)
python -c "
from moshi.models.loaders import CheckpointInfo
info = CheckpointInfo.from_hf_repo('kyutai/stt-1b-en_fr')
print('STT model downloaded:', info)
"

# TTS 1.6B
python -c "
from moshi.models.loaders import CheckpointInfo
info = CheckpointInfo.from_hf_repo('kyutai/tts-1.6b-en_fr')
print('TTS model downloaded:', info)
"
```

### 1.3 — Valider STT en isolation

```bash
# Tester avec un fichier audio
python -m moshi.run_inference --hf-repo kyutai/stt-1b-en_fr test_audio.wav
```

**Critere de succes** : Le texte transcrit correspond a l'audio d'entree.

### 1.4 — Valider TTS en isolation

```bash
# Tester la synthese vocale
echo "Bonjour, je suis AudioClaude" | python scripts/tts_pytorch.py - test_output.wav
# Ecouter le resultat
afplay test_output.wav
```

**Critere de succes** : L'audio de sortie est naturel et intelligible.

### 1.5 — Valider Mimi en isolation

```python
import torch
from moshi.models import loaders

mimi_weight = loaders.get_mimi_weight()
mimi = loaders.get_mimi(mimi_weight, device='cpu')

# Encoder puis decoder un signal test
wav = torch.randn(1, 1, 24000 * 3)  # 3 secondes
codes = mimi.encode(wav)
decoded = mimi.decode(codes)
print(f"Input shape: {wav.shape}")
print(f"Codes shape: {codes.shape}")  # [1, K, T] a 12.5 Hz
print(f"Decoded shape: {decoded.shape}")
print(f"Compression: {wav.numel() / codes.numel():.0f}x")
```

**Critere de succes** : Encode/decode sans erreur, ratio de compression ~200x.

### 1.6 — Valider tmux

```bash
# Creer une session de test
tmux new-session -d -s audioclaude-test

# Injecter une commande
tmux send-keys -t audioclaude-test "echo hello from AudioClaude" Enter

# Capturer la sortie
tmux capture-pane -t audioclaude-test -p
```

**Critere de succes** : La commande est injectee et la sortie capturee correctement.

---

## Checklist de validation Phase 1

- [ ] Python 3.12+ installe et venv actif
- [ ] `moshi>=0.2.11` installe
- [ ] `rustymimi` installe
- [ ] Modele STT 1B telecharge et fonctionnel
- [ ] Modele TTS 1.6B telecharge et fonctionnel
- [ ] Mimi encode/decode fonctionne
- [ ] tmux send-keys et capture-pane fonctionnent
- [ ] Claude Code CLI installe et accessible dans le PATH

---

## Hardware : Apple Silicon vs NVIDIA

### Sur Apple Silicon (MLX)

```bash
pip install "moshi_mlx>=0.2.12"

# STT via MLX
python -m moshi_mlx.run_inference --hf-repo kyutai/stt-2.6b-en-mlx audio.wav --temp 0

# TTS via MLX (quantize pour la vitesse)
echo "Hello" | python scripts/tts_mlx.py - - --quantize 8
```

### Sur NVIDIA GPU

```bash
# Verifier CUDA
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name())"

# Les modeles PyTorch standard fonctionnent directement
# STT et TTS utilisent device='cuda' automatiquement
```

---

## Structure des fichiers a creer dans cette phase

```
server/
├── requirements.txt          # Toutes les dependances pip
├── pyproject.toml            # Metadata projet
└── tests/
    ├── test_stt.py           # Test STT en isolation
    ├── test_tts.py           # Test TTS en isolation
    └── test_mimi.py          # Test Mimi encode/decode
```
