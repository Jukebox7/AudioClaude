# AudioClaude — Plan de projet

> **Objectif** : Construire une interface vocale mobile (iPhone) vers un Mac distant executant Claude Code dans tmux, avec voix naturelle Kyutai (TTS/STT) et codec Mimi ultra-basse bande passante, le tout securise par SSH.

---

## Vue d'ensemble

AudioClaude transforme un iPhone en terminal vocal distant pour Claude Code. L'utilisateur peut partir faire ses courses, dicter des directives a Claude, entendre ses reponses en voix naturelle, et retrouver sa session intacte au retour.

---

## Architecture globale

```mermaid
graph TB
    subgraph iPhone["iPhone (Client leger)"]
        MIC[Microphone / AirPods]
        SPK[Haut-parleur / AirPods]
        APP[App Swift - AudioClaude]
        MIC --> APP
        APP --> SPK
    end

    subgraph SSH_Tunnel["Tunnel SSH chiffre"]
        UP["Audio encode Mimi (1.1 kbps)"]
        DOWN["Audio encode Mimi (1.1 kbps)"]
    end

    subgraph Mac["Mac (Serveur GPU)"]
        MIMI_DEC["Mimi Decode"]
        STT["Kyutai STT (1B)"]
        BRIDGE["Bridge audioclaude-server"]
        TMUX["tmux session"]
        CC["Claude Code"]
        TTS["Kyutai TTS (1.6B)"]
        MIMI_ENC["Mimi Encode"]

        MIMI_DEC --> STT
        STT --> BRIDGE
        BRIDGE -->|send-keys| TMUX
        TMUX --- CC
        TMUX -->|capture-pane| BRIDGE
        BRIDGE --> TTS
        TTS --> MIMI_ENC
    end

    APP -->|Upload| UP
    UP --> MIMI_DEC
    MIMI_ENC --> DOWN
    DOWN -->|Download| APP

    style iPhone fill:#1a1a2e,stroke:#e94560,color:#fff
    style Mac fill:#1a1a2e,stroke:#0f3460,color:#fff
    style SSH_Tunnel fill:#16213e,stroke:#e94560,color:#fff
```

---

## Flux de donnees detaille

```mermaid
sequenceDiagram
    participant U as Utilisateur (iPhone)
    participant APP as App Swift
    participant SSH as Tunnel SSH
    participant SRV as audioclaude-server (Mac)
    participant STT as Kyutai STT
    participant TMUX as tmux + Claude Code
    participant TTS as Kyutai TTS

    Note over U,TTS: --- Phase 1 : Commande vocale ---
    U->>APP: Parle dans le micro
    APP->>APP: Capture audio PCM 24kHz
    APP->>APP: Mimi encode (1.1 kbps)
    APP->>SSH: Stream tokens Mimi
    SSH->>SRV: Tokens Mimi chiffres
    SRV->>SRV: Mimi decode -> PCM
    SRV->>STT: Audio PCM
    STT->>SRV: Texte transcrit + timestamps
    SRV->>SRV: Detecte fin de parole (VAD)

    Note over U,TTS: --- Phase 2 : Injection dans Claude ---
    SRV->>TMUX: tmux send-keys "texte transcrit"
    TMUX->>TMUX: Claude Code traite la commande

    Note over U,TTS: --- Phase 3 : Retour vocal ---
    loop Polling toutes les 2s
        SRV->>TMUX: tmux capture-pane -p
        TMUX-->>SRV: Contenu du terminal
    end
    SRV->>SRV: Detecte nouveau contenu
    SRV->>TTS: Nouveau texte a lire
    TTS->>SRV: Audio PCM synthetise
    SRV->>SRV: Mimi encode
    SRV->>SSH: Stream tokens Mimi
    SSH->>APP: Tokens Mimi
    APP->>APP: Mimi decode -> PCM
    APP->>U: Audio dans AirPods
```

---

## Composants du systeme

```mermaid
graph LR
    subgraph Core["Composants principaux"]
        A["audioclaude-server<br/>(Python)"]
        B["audioclaude-client<br/>(Swift/iOS)"]
    end

    subgraph Kyutai["Stack Kyutai"]
        C["Mimi Codec<br/>rustymimi"]
        D["Kyutai STT 1B<br/>moshi>=0.2.6"]
        E["Kyutai TTS 1.6B<br/>moshi==0.2.11"]
    end

    subgraph Infra["Infrastructure"]
        F["SSH Tunnel"]
        G["tmux"]
        H["Claude Code CLI"]
    end

    A --> C
    A --> D
    A --> E
    A --> F
    A --> G
    B --> C
    G --> H

    style Core fill:#0f3460,stroke:#e94560,color:#fff
    style Kyutai fill:#533483,stroke:#e94560,color:#fff
    style Infra fill:#16213e,stroke:#0f3460,color:#fff
```

---

## Diagramme des etats du systeme

```mermaid
stateDiagram-v2
    [*] --> Idle: Demarrage serveur

    Idle --> Listening: Connexion iPhone
    Listening --> Transcribing: Audio detecte (VAD)
    Transcribing --> Processing: Fin de parole
    Processing --> Injecting: Texte valide
    Injecting --> WaitingClaude: send-keys envoye
    WaitingClaude --> Reading: Claude repond
    Reading --> Speaking: Nouveau contenu detecte
    Speaking --> Listening: TTS termine

    Listening --> Idle: Deconnexion iPhone
    WaitingClaude --> Listening: Timeout (Claude travaille en silence)

    state Listening {
        [*] --> CaptureAudio
        CaptureAudio --> MimiDecode
        MimiDecode --> VADCheck
        VADCheck --> CaptureAudio: Silence
        VADCheck --> [*]: Parole detectee
    }

    state Speaking {
        [*] --> CapturePaneOutput
        CapturePaneOutput --> FilterNew
        FilterNew --> TTSSynth
        TTSSynth --> MimiEncode
        MimiEncode --> StreamToiPhone
        StreamToiPhone --> [*]
    }
```

---

## Architecture reseau

```mermaid
graph TB
    subgraph iPhone_Net["iPhone (4G/WiFi)"]
        C1["Client SSH<br/>Port dynamique"]
    end

    subgraph Tunnel["Tunnel SSH"]
        P1["Port 9000<br/>Audio UP (iPhone->Mac)<br/>Mimi tokens"]
        P2["Port 9001<br/>Audio DOWN (Mac->iPhone)<br/>Mimi tokens"]
        P3["Port 9002<br/>Controle / Status<br/>JSON WebSocket"]
    end

    subgraph Mac_Net["Mac (LAN/localhost)"]
        S1["audioclaude-server<br/>:9000 :9001 :9002"]
        S2["tmux socket"]
        S3["Claude Code"]
    end

    C1 -->|"ssh -L 9000:localhost:9000<br/>-L 9001:localhost:9001<br/>-L 9002:localhost:9002"| S1
    S1 --- S2
    S2 --- S3

    style iPhone_Net fill:#1a1a2e,stroke:#e94560,color:#fff
    style Mac_Net fill:#1a1a2e,stroke:#0f3460,color:#fff
    style Tunnel fill:#16213e,stroke:#e94560,color:#fff
```

---

## Stack technique

| Composant | Technologie | Version | Role |
|-----------|------------|---------|------|
| STT | Kyutai STT | 1B en_fr | Transcription voix -> texte |
| TTS | Kyutai TTS | 1.6B en_fr | Synthese texte -> voix naturelle |
| Codec | Mimi (rustymimi) | >=0.2.6 | Compression audio 1.1 kbps |
| Serveur | Python + asyncio | 3.12+ | Orchestration des composants |
| Client iOS | Swift + AVFoundation | iOS 17+ | Capture/lecture audio |
| Transport | SSH tunnel | OpenSSH | Chiffrement de bout en bout |
| Session | tmux | 3.x | Persistance terminal |
| IA | Claude Code CLI | latest | Assistant de dev |

---

## Plan de realisation par phases

| Phase | Fichier | Description | Duree estimee |
|-------|---------|-------------|---------------|
| 0 | [00-architecture.md](00-architecture.md) | Architecture et decisions techniques | - |
| 1 | [01-serveur-mac.md](01-serveur-mac.md) | Setup serveur Mac + dependances Kyutai | Phase 1 |
| 2 | [02-pipeline-audio.md](02-pipeline-audio.md) | Pipeline STT + TTS + Mimi | Phase 2 |
| 3 | [03-tmux-bridge.md](03-tmux-bridge.md) | Integration tmux <-> Claude Code | Phase 3 |
| 4 | [04-transport-ssh.md](04-transport-ssh.md) | Protocole de transport SSH + WebSocket | Phase 4 |
| 5 | [05-client-iphone.md](05-client-iphone.md) | App iOS Swift | Phase 5 |
| 6 | [06-integration.md](06-integration.md) | Tests d'integration bout-en-bout | Phase 6 |
| 7 | [07-ux-polish.md](07-ux-polish.md) | UX, robustesse, mode offline | Phase 7 |

---

## Prerequis materiel

- **Mac** : Apple Silicon (M1+) ou GPU NVIDIA (>=12 Go VRAM)
  - Pour STT 1B + TTS 1.6B en simultane : 16+ Go VRAM recommande
  - MLX sur Apple Silicon : pas besoin de GPU discrete
- **iPhone** : iPhone 12+ avec iOS 17+
- **Reseau** : WiFi local ou 4G/5G pour le mode mobile
  - Bande passante requise : ~3 kbps (2x 1.1 kbps Mimi + controle)

---

## Architecture du projet

```mermaid
graph TB
    subgraph Root["moshi/"]
        CLAUDE["CLAUDE.md<br/><i>Config Claude Code</i>"]

        subgraph Plans["plans/"]
            direction TB
            README_P["README.md<br/><i>Ce fichier</i>"]
            P0["00-architecture.md"]
            P1["01-serveur-mac.md"]
            P2["02-pipeline-audio.md"]
            P3["03-tmux-bridge.md"]
            P4["04-transport-ssh.md"]
            P5["05-client-iphone.md"]
            P6["06-integration.md"]
            P7["07-ux-polish.md"]
        end

        subgraph Server["server/ — Python (Mac)"]
            direction TB
            subgraph AudioClaudeModule["audioclaude/"]
                direction TB
                INIT["__init__.py"]
                SRV["server.py<br/><i>WebSocket asyncio</i>"]
                STT_M["stt.py<br/><i>Kyutai STT 1B</i>"]
                TTS_M["tts.py<br/><i>Kyutai TTS 1.6B</i>"]
                MIMI_M["mimi_codec.py<br/><i>rustymimi</i>"]
                TMUX_M["tmux_bridge.py<br/><i>send-keys / capture-pane</i>"]
                FILTER["output_filter.py<br/><i>Filtre pour TTS</i>"]
                STATE["claude_state.py<br/><i>Detection etat Claude</i>"]
                PROTO["protocol.py<br/><i>Messages WebSocket</i>"]
            end
            REQ["requirements.txt"]
            PYPROJ["pyproject.toml"]
            subgraph Tests["tests/"]
                T_STT["test_stt.py"]
                T_TTS["test_tts.py"]
                T_MIMI["test_mimi.py"]
                T_BRIDGE["test_tmux_bridge.py"]
                T_FULL["test_full_local.py"]
                T_WS["test_ws_client.py"]
            end
        end

        subgraph IOS["ios/ — Swift (iPhone)"]
            direction TB
            subgraph XCODE["AudioClaude/"]
                XCPROJ["AudioClaude.xcodeproj"]
                subgraph Sources["Sources/"]
                    direction TB
                    APP_S["App.swift<br/><i>Point d'entree</i>"]
                    CV["ContentView.swift<br/><i>UI principale</i>"]
                    AC["AudioCapture.swift<br/><i>AVAudioEngine</i>"]
                    AP["AudioPlayback.swift<br/><i>AVAudioPlayerNode</i>"]
                    WSC["WebSocketClient.swift<br/><i>URLSession WS</i>"]
                    SSHT["SSHTransport.swift<br/><i>Tunnel SSH</i>"]
                    PTT["PushToTalkButton.swift"]
                    SB["StatusBar.swift"]
                end
                RES["Resources/<br/><i>Info.plist, assets</i>"]
            end
        end

        subgraph Scripts["scripts/"]
            S1["setup-mac.sh<br/><i>Install deps</i>"]
            S2["start-server.sh<br/><i>Lance serveur</i>"]
            S3["connect-iphone.sh<br/><i>Tunnel SSH</i>"]
        end
    end

    %% Relations entre modules serveur
    SRV --> STT_M
    SRV --> TTS_M
    SRV --> MIMI_M
    SRV --> TMUX_M
    SRV --> PROTO
    TMUX_M --> FILTER
    TMUX_M --> STATE

    %% Relations iPhone
    CV --> AC
    CV --> AP
    CV --> WSC
    WSC --> SSHT

    style Root fill:#0d1117,stroke:#30363d,color:#e6edf3
    style Plans fill:#161b22,stroke:#1f6feb,color:#e6edf3
    style Server fill:#161b22,stroke:#3fb950,color:#e6edf3
    style AudioClaudeModule fill:#1c2128,stroke:#3fb950,color:#e6edf3
    style Tests fill:#1c2128,stroke:#3fb950,color:#e6edf3
    style IOS fill:#161b22,stroke:#f78166,color:#e6edf3
    style XCODE fill:#1c2128,stroke:#f78166,color:#e6edf3
    style Sources fill:#21262d,stroke:#f78166,color:#e6edf3
    style Scripts fill:#161b22,stroke:#d29922,color:#e6edf3
```

## Arborescence cible (texte)

```
moshi/
├── CLAUDE.md                          # Config Claude Code
├── plans/                             # Documentation et plans
│   ├── README.md                      # Ce fichier
│   └── 00 a 07-*.md                   # Plans par phase
├── server/                            # Serveur Mac (Python)
│   ├── audioclaude/                   # Package principal
│   │   ├── server.py                  # WebSocket asyncio
│   │   ├── stt.py                     # Kyutai STT
│   │   ├── tts.py                     # Kyutai TTS
│   │   ├── mimi_codec.py              # Mimi encode/decode
│   │   ├── tmux_bridge.py             # Interface tmux
│   │   ├── output_filter.py           # Filtre contenu TTS
│   │   ├── claude_state.py            # Detection etat Claude
│   │   └── protocol.py                # Protocole WebSocket
│   ├── tests/                         # Tests unitaires + integration
│   ├── requirements.txt
│   └── pyproject.toml
├── ios/                               # App iPhone (Swift)
│   └── AudioClaude/
│       ├── AudioClaude.xcodeproj
│       ├── Sources/                   # Code Swift
│       └── Resources/                 # Assets, Info.plist
└── scripts/                           # Scripts utilitaires
    ├── setup-mac.sh
    ├── start-server.sh
    └── connect-iphone.sh
```
