 Arsitektur yang didokumentasikan:

```
Arch Linux
├── /opt/llama.cpp
│   └── llama-server (Vulkan)
│
├── /opt/models/
│   ├── qwen3/
│   │   └── Qwen3-8B-Q4_K_M.gguf
│   ├── qwen2.5-vl-7b/
│   │   ├── Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf
│   │   └── mmproj-*.gguf
│   └── current.gguf -> model aktif
│
└── systemd
    └── llama-qwen.service
```

 `llama-server` memang menyediakan WebUI HTTP, Vulkan adalah salah satu backend GPU llama.cpp, dan model multimodal lokal menggunakan `--mmproj` bersama file model utama.  GitHub+2

---

 # LLM Homelab Manager

 Simple LLM server manager for **Arch Linux + llama.cpp + Vulkan**, designed for a headless homelab.

 The server exposes llama.cpp's WebUI over the LAN, so other devices only need a browser to start chatting.

 ## Features

 - AMD GPU acceleration through Vulkan
- Designed for headless Arch Linux servers
- `systemd` managed llama-server
- One-command model switching
- Hugging Face GGUF downloading
- Automatic detection of multimodal `mmproj`
- Separate model directories
- RAM protection through systemd cgroup limits
- Manual start/stop
- WebUI accessible from other devices on the LAN

 ## Tested Hardware

 Example setup:

 - CPU: x86\_64
- GPU: AMD Radeon RX 6600 XT 8 GB
- RAM: 16 GB
- OS: Arch Linux
- GPU backend: Vulkan / RADV
- Inference engine: llama.cpp
- Server port: `18080`

 The exact hardware is not required. Any Vulkan-capable GPU supported by llama.cpp can be used.

---

 # Architecture

```
                  LAN
                   │
          ┌────────┴────────┐
          │                 │
       Laptop             Phone
          │                 │
          └────────┬────────┘
                   │
             HTTP :18080
                   │
                   ▼
        ┌─────────────────────┐
        │     llama-server    │
        │                     │
        │   Vulkan backend    │
        └──────────┬──────────┘
                   │
                   ▼
             AMD GPU / VRAM

/opt/models/
│
├── qwen3/
│   └── Qwen3-8B-Q4_K_M.gguf
│
├── qwen2.5-vl-7b/
│   ├── model-Q4_K_M.gguf
│   └── mmproj-f16.gguf
│
└── current.gguf
        │
        └── symlink → active model
```

 Only one model is loaded at a time.

 This is intentional for systems with limited RAM/VRAM.

---

 # 1\. Install dependencies

 Update Arch:

```
sudo pacman -Syu
```

 Install build dependencies and Vulkan:

```
sudo pacman -S --needed \
    base-devel \
    cmake \
    git \
    vulkan-headers \
    vulkan-radeon \
    vulkan-tools \
    python-huggingface-hub
```

 Verify Vulkan:

```
vulkaninfo --summary
```

 A working AMD system should show something similar to:

```
GPU0:
    deviceName = AMD Radeon RX 6600 XT
    driverName = radv
```

---

 # 2\. Build llama.cpp with Vulkan

 Clone llama.cpp:

```
cd /opt

sudo git clone https://github.com/ggml-org/llama.cpp.git

sudo chown -R "$USER:$USER" /opt/llama.cpp
```

 Build:

```
cd /opt/llama.cpp

cmake -B build \
    -DGGML_VULKAN=ON \
    -DCMAKE_BUILD_TYPE=Release

cmake --build build --config Release -j$(nproc)
```

 The official llama.cpp documentation uses CMake and `GGML_VULKAN=ON` for Vulkan builds.  GitHub

 Verify:

```
./build/bin/llama-server --list-devices
```

 Example:

```
Available devices:
  Vulkan0: AMD Radeon RX 6600 XT (RADV NAVI23) (8192 MiB, ...)
```

 If `Vulkan0` is shown, llama.cpp can see the GPU.

---

 # 3\. Create model directory

```
sudo mkdir -p /opt/models
sudo chown -R "$USER:$USER" /opt/models
```

 The directory is organized by model:

```
/opt/models/
├── qwen3/
├── qwen2.5-vl-7b/
└── ...
```

 The active model is represented by:

```
/opt/models/current.gguf
```

 This is a symbolic link, not another copy of the model.

---

 # 4\. Hugging Face CLI

 The project uses the official `hf` CLI from `huggingface_hub`.

 Check:

```
hf --help
```

 The modern Hugging Face CLI is named `hf`, and `hf download` supports downloading files from a repository directly into a local directory.  Hugging Face+1

---

 # 5\. systemd service

 Create:

```
sudo nano /etc/systemd/system/llama-qwen.service
```

 Use:

```
[Unit]
Description=LLM Server - llama.cpp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple

User=server
WorkingDirectory=/opt/llama.cpp

ExecStart=/opt/llama.cpp/build/bin/llama-server \
    -m /opt/models/current.gguf \
    --device Vulkan0 \
    -ngl 99 \
    -c 8192 \
    --host 0.0.0.0 \
    --port 18080

Restart=on-failure
RestartSec=5

# Protect a 16 GB system from the LLM consuming all RAM.
MemoryHigh=7G
MemoryMax=9G
OOMScoreAdjust=500

[Install]
WantedBy=multi-user.target
```

 Reload systemd:

```
sudo systemctl daemon-reload
```

 The service is intentionally **not enabled at boot**.

 The LLM is started manually using:

```
lm-start
```

 This keeps RAM available when the LLM is not needed.

---

 # 6\. lm-list

 `lm-list` displays all locally installed models.

 Create:

```
sudo nano /usr/local/bin/lm-list
```

```
#!/bin/sh

MODEL_DIR="/opt/models"
CURRENT="$MODEL_DIR/current.gguf"

echo "Available models:"
echo

found=0

for dir in "$MODEL_DIR"/*/; do
    [ -d "$dir" ] || continue

    model=$(find "$dir" -maxdepth 1 -type f -name '*.gguf' \
        ! -name 'mmproj*.gguf' \
        -print -quit)

    [ -n "$model" ] || continue

    name=$(basename "$dir")

    if [ -L "$CURRENT" ] &&
       [ "$(readlink -f "$CURRENT")" = "$(readlink -f "$model")" ]; then

        printf "  * %-28s %s\n" \
            "$name" \
            "$(basename "$model")"
    else
        printf "    %-28s %s\n" \
            "$name" \
            "$(basename "$model")"
    fi

    found=1
done

if [ "$found" -eq 0 ]; then
    echo "  (belum ada model)"
fi
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-list
```

 Example:

```
lm-list
```

 Output:

```
Available models:

  * qwen3                       Qwen3-8B-Q4_K_M.gguf
    qwen2.5-vl-7b              Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf
```

 `*` means the model is currently active.

---

 # 7\. lm-current

 Displays the active model.

```
sudo nano /usr/local/bin/lm-current
```

```
#!/bin/sh

CURRENT="/opt/models/current.gguf"

if [ -L "$CURRENT" ] && [ -e "$CURRENT" ]; then
    MODEL=$(readlink -f "$CURRENT")

    echo "Current model:"
    echo "  $(basename "$(dirname "$MODEL")")"
    echo
    echo "File:"
    echo "  $(basename "$MODEL")"
    echo
    echo "Path:"
    echo "  $MODEL"
else
    echo "Belum ada model aktif."
    exit 1
fi
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-current
```

 Usage:

```
lm-current
```

---

 # 8\. lm-download

 `lm-download` downloads GGUF models from Hugging Face.

 Usage:

```
lm-download <huggingface-repository>
```

 Example:

```
lm-download Qwen/Qwen3-8B-GGUF
```

 The script prefers `Q4_K_M` when available.

 For multimodal repositories, it also downloads the `mmproj` file.

 Create:

```
sudo nano /usr/local/bin/lm-download
```

```
#!/bin/sh

MODEL_DIR="/opt/models"

if [ -z "$1" ]; then
    echo "Usage:"
    echo "  lm-download <huggingface-repo>"
    echo
    echo "Example:"
    echo "  lm-download Qwen/Qwen3-8B-GGUF"
    exit 1
fi

REPO="$1"
NAME=$(basename "$REPO")
TARGET="$MODEL_DIR/$NAME"

if ! command -v hf >/dev/null 2>&1; then
    echo "Error: 'hf' command not found."
    echo
    echo "Install:"
    echo "  sudo pacman -S python-huggingface-hub"
    exit 1
fi

mkdir -p "$TARGET"

echo "==> Checking Hugging Face repo:"
echo "    $REPO"
echo

echo "==> Querying repository..."

FILES=$(
    curl -fsSL \
        "https://huggingface.co/api/models/$REPO/tree/main?recursive=true" |
    grep -o '"path":"[^"]*\.gguf"' |
    sed 's/"path":"//;s/"$//' |
    sort
)

if [ -z "$FILES" ]; then
    echo "No GGUF files found."
    echo
    echo "This repository may not contain GGUF files."
    exit 1
fi

echo "Available GGUF files:"
printf '%s\n' "$FILES"
echo

# Prefer Q4_K_M as the default quantization.
MODEL=$(
    printf '%s\n' "$FILES" |
    grep -E 'Q4_K_M\.gguf$' |
    grep -vi 'mmproj' |
    head -n 1
)

# Fallback to Q4_K_S.
if [ -z "$MODEL" ]; then
    MODEL=$(
        printf '%s\n' "$FILES" |
        grep -E 'Q4_K_S\.gguf$' |
        grep -vi 'mmproj' |
        head -n 1
    )
fi

# Final fallback: first non-mmproj GGUF.
if [ -z "$MODEL" ]; then
    MODEL=$(
        printf '%s\n' "$FILES" |
        grep -vi 'mmproj' |
        head -n 1
    )
fi

if [ -z "$MODEL" ]; then
    echo "No main GGUF model found."
    exit 1
fi

echo "==> Selected model:"
echo "    $MODEL"
echo

echo "==> Downloading model..."

hf download "$REPO" "$MODEL" \
    --local-dir "$TARGET" || exit 1

# Detect multimodal projector.
MMPROJ=$(
    printf '%s\n' "$FILES" |
    grep -Ei '(^|/)mmproj.*\.gguf$' |
    head -n 1
)

if [ -n "$MMPROJ" ]; then
    echo
    echo "==> Multimodal projector detected:"
    echo "    $MMPROJ"
    echo

    echo "==> Downloading projector..."

    hf download "$REPO" "$MMPROJ" \
        --local-dir "$TARGET" || exit 1
fi

echo
echo "========================================"
echo "Download complete"
echo "========================================"
echo
echo "Model directory:"
echo "  $TARGET"
echo
echo "Run:"
echo "  lm-list"
echo
echo "Then:"
echo "  lm-use $NAME"
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-download
```

 Example:

```
lm-download Qwen/Qwen3-8B-GGUF
```

 For a multimodal model:

```
lm-download <owner>/<vision-model-GGUF-repo>
```

 The llama.cpp multimodal system uses a main GGUF plus an `mmproj` projector for local multimodal inference.  GitHub

---

 # 9\. lm-use

 `lm-use` switches the active model.

 It accepts:

 - model directory name
- exact GGUF filename
- partial GGUF filename

 Examples:

```
lm-use qwen3
```

 or:

```
lm-use Qwen3-8B-Q4_K_M.gguf
```

 The active model is stored as:

```
/opt/models/current.gguf
```

 Create:

```
sudo nano /usr/local/bin/lm-use
```

```
#!/bin/sh

MODEL_DIR="/opt/models"
CURRENT="$MODEL_DIR/current.gguf"

if [ -z "$1" ]; then
    echo "Usage:"
    echo "  lm-use <model>"
    echo
    lm-list
    exit 1
fi

QUERY="$1"
MODEL=""
DIR=""

# Exact directory match.
if [ -d "$MODEL_DIR/$QUERY" ]; then
    DIR="$MODEL_DIR/$QUERY"

    MODEL=$(
        find "$DIR" \
            -maxdepth 1 \
            -type f \
            -name '*.gguf' \
            ! -name 'mmproj*.gguf' \
            -print -quit
    )
fi

# Exact filename match.
if [ -z "$MODEL" ]; then
    MODEL=$(
        find "$MODEL_DIR" \
            -type f \
            -name "$QUERY" \
            ! -name 'mmproj*.gguf' \
            -print -quit 2>/dev/null
    )

    if [ -n "$MODEL" ]; then
        DIR=$(dirname "$MODEL")
    fi
fi

# Partial filename match.
if [ -z "$MODEL" ]; then
    MODEL=$(
        find "$MODEL_DIR" \
            -type f \
            -name "*$QUERY*" \
            -name '*.gguf' \
            ! -name 'mmproj*.gguf' \
            -print -quit 2>/dev/null
    )

    if [ -n "$MODEL" ]; then
        DIR=$(dirname "$MODEL")
    fi
fi

if [ -z "$MODEL" ]; then
    echo "Model '$QUERY' tidak ditemukan."
    echo
    lm-list
    exit 1
fi

# Stop server before changing the active model.
if systemctl is-active --quiet llama-qwen.service; then
    echo "LLM sedang berjalan."
    echo "Stopping..."
    sudo systemctl stop llama-qwen.service
fi

ln -sfn "$MODEL" "$CURRENT"

echo
echo "Model aktif:"
echo "  Folder : $(basename "$DIR")"
echo "  File   : $(basename "$MODEL")"

# Detect mmproj.
MMPROJ=$(
    find "$DIR" \
        -maxdepth 1 \
        -type f \
        -name 'mmproj*.gguf' \
        -print -quit
)

if [ -n "$MMPROJ" ]; then
    echo "  Vision : $(basename "$MMPROJ")"
fi

echo
echo "Jalankan:"
echo "  lm-start"
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-use
```

---

 # 10\. Multimodal models

 A multimodal model can contain:

```
model.gguf
mmproj-f16.gguf
```

 The main model contains the language model.

 The `mmproj` provides the multimodal projector.

 llama.cpp supports loading them with:

```
llama-server \
    -m model.gguf \
    --mmproj mmproj-f16.gguf
```

 The projector can also be offloaded to the GPU.  GitHub+1

 The manager detects `mmproj*.gguf` automatically.

 Therefore:

```
lm-use qwen2.5-vl-7b
```

 can identify:

```
model:
    Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf

projector:
    mmproj-f16.gguf
```

 The systemd service should then include the projector automatically.

---

 # 11\. lm-start

 `lm-start` checks whether a model is selected before starting systemd.

 Create:

```
sudo nano /usr/local/bin/lm-start
```

```
#!/bin/sh

CURRENT="/opt/models/current.gguf"

if [ ! -L "$CURRENT" ] || [ ! -e "$CURRENT" ]; then
    echo "Belum ada model aktif."
    echo
    echo "Gunakan:"
    echo "  lm-list"
    echo "  lm-use <model>"
    exit 1
fi

if systemctl is-active --quiet llama-qwen.service; then
    IP=$(
        ip -4 addr show scope global |
        awk '/inet / {
            sub("/.*", "", $2)
            print $2
            exit
        }'
    )

    echo "LLM sudah berjalan."
    echo "WebUI: http://${IP}:18080"
    exit 0
fi

sudo systemctl start llama-qwen.service

sleep 2

if systemctl is-active --quiet llama-qwen.service; then
    IP=$(
        ip -4 addr show scope global |
        awk '/inet / {
            sub("/.*", "", $2)
            print $2
            exit
        }'
    )

    echo "LLM started."
    echo
    echo "Model:"
    echo "  $(basename "$(readlink -f "$CURRENT")")"
    echo
    echo "WebUI:"
    echo "  http://${IP}:18080"
else
    echo "LLM gagal start."
    echo
    sudo systemctl status llama-qwen.service --no-pager
    exit 1
fi
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-start
```

 Usage:

```
lm-start
```

---

 # 12\. lm-stop

 Create:

```
sudo nano /usr/local/bin/lm-stop
```

```
#!/bin/sh

if systemctl is-active --quiet llama-qwen.service; then
    sudo systemctl stop llama-qwen.service
    echo "LLM stopped."
else
    echo "LLM sudah berhenti."
fi
```

 Make executable:

```
sudo chmod +x /usr/local/bin/lm-stop
```

 Usage:

```
lm-stop
```

 Stopping the service releases the model's RAM and VRAM allocations.

---

 # 13\. Final command set

 The complete interface is:

```
lm-list
lm-current
lm-download <repo>
lm-use <model>
lm-start
lm-stop
```

 ## List models

```
lm-list
```

 ## Show current model

```
lm-current
```

 ## Download model

```
lm-download Qwen/Qwen3-8B-GGUF
```

 ## Select model

```
lm-use qwen3
```

 ## Start server

```
lm-start
```

 ## Stop server

```
lm-stop
```

---

 # 14\. Typical workflow

 First installation:

```
lm-download Qwen/Qwen3-8B-GGUF
lm-list
lm-use qwen3
lm-start
```

 Open from another device:

```
http://SERVER-IP:18080
```

 Switch model:

```
lm-use another-model
lm-start
```

 Stop when finished:

```
lm-stop
```

---

 # 15\. RAM management

 For a 16 GB RAM machine, the example service uses:

```
MemoryHigh=7G
MemoryMax=9G
```

 This leaves approximately 7 GB for the rest of the system under normal conditions.

 The LLM may still use GPU VRAM independently.

 Check current memory:

```
systemctl show llama-qwen.service \
    -p MemoryCurrent \
    -p MemoryHigh \
    -p MemoryMax
```

 Check service status:

```
systemctl status llama-qwen.service
```

 View logs:

```
journalctl -u llama-qwen.service -f
```

 If the model requires more system RAM, adjust `MemoryMax`.

 For an RX 6600 XT 8 GB + 16 GB RAM machine, starting with an 8K context is a conservative baseline.

---

 # 16\. GPU verification

 List llama.cpp devices:

```
/opt/llama.cpp/build/bin/llama-server --list-devices
```

 Expected:

```
Available devices:
  Vulkan0: AMD Radeon RX 6600 XT (RADV NAVI23)
```

 The server explicitly selects:

```
--device Vulkan0
```

 and requests GPU layer offloading:

```
-ngl 99
```

 llama.cpp supports Vulkan as a GPU backend and exposes device selection through `--device`.  GitHub+1

---

 # 17\. Troubleshooting

 ## `Belum ada model aktif`

 Run:

```
lm-list
```

 Then:

```
lm-use <model>
```

 Then:

```
lm-start
```

---

 ## Port 18080 already in use

 Check:

```
ss -ltnp | grep ':18080'
```

 Change the service:

```
sudo nano /etc/systemd/system/llama-qwen.service
```

 Change:

```
--port 18080
```

 to another port, then:

```
sudo systemctl daemon-reload
```

---

 ## GPU not detected

 Run:

```
vulkaninfo --summary
```

 Then:

```
/opt/llama.cpp/build/bin/llama-server --list-devices
```

 If the AMD GPU does not appear, verify:

```
pacman -Qs vulkan-radeon
```

 and:

```
pacman -Qs mesa
```

---

 ## Model fails to load

 Check:

```
journalctl -u llama-qwen.service -n 100 --no-pager
```

 Also verify:

```
ls -lh /opt/models/current.gguf
```

 and:

```
readlink -f /opt/models/current.gguf
```

---

 ## Multimodal model does not accept images

 Verify that the model directory contains both:

```
model-Q4_K_M.gguf
mmproj-*.gguf
```

 For manual testing:

```
llama-server \
    -m /path/to/model.gguf \
    --mmproj /path/to/mmproj-f16.gguf
```

 llama.cpp's multimodal server supports image input through its chat interface/API.  GitHub

---

 # 18\. Security

 The server is configured with:

```
--host 0.0.0.0
```

 This makes the WebUI accessible from the LAN.

 Do **not** expose port `18080` directly to the public Internet without authentication and appropriate network controls.

 For a home LAN, restrict access with the firewall to your trusted subnet.

 Do not enable llama.cpp's filesystem/agent tools unless you specifically understand the security implications. llama.cpp documents that features involving filesystem access should be treated carefully and recommends explicit CORS/API-key configuration for exposed deployments.  GitHub

---

 # 19\. Directory layout

 Final installation:

```
/opt/
├── llama.cpp/
│   ├── build/
│   │   └── bin/
│   │       └── llama-server
│   └── ...
│
└── models/
    ├── current.gguf
    │
    ├── qwen3/
    │   └── Qwen3-8B-Q4_K_M.gguf
    │
    └── qwen2.5-vl-7b/
        ├── Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf
        └── mmproj-f16.gguf
```

---

 # 20\. Design philosophy

 This project intentionally keeps model management separate from inference.

 `llama-server` handles:

 - inference
- GPU acceleration
- WebUI
- API
- multimodal inference

 The `lm-*` commands handle:

 - downloading models
- selecting models
- starting/stopping inference

 This means replacing llama.cpp later does not require redesigning the model manager.

---

 # 21\. Quick reference

 | Command | Function |
| --- | --- |
| `lm-list` | List installed models |
| `lm-current` | Show active model |
| `lm-download REPO` | Download GGUF from Hugging Face |
| `lm-use MODEL` | Select model |
| `lm-start` | Start LLM server |
| `lm-stop` | Stop LLM server |

Typical session:

```
lm-list
lm-use qwen3
lm-start
```

 Then open:

```
http://SERVER-IP:18080
```

 When finished:

```
lm-stop
```

---

 # Credits

 This project uses:

 - [llama.cpp](<https://github.com/ggml-org/llama.cpp>) for local LLM inference and WebUI
- [Hugging Face Hub](<https://huggingface.co/>) for model distribution
- Vulkan / Mesa RADV for AMD GPU acceleration

 llama.cpp is an open-source LLM inference project with support for multiple hardware backends including Vulkan.  GitHub
