# development AI setup guide

environment setup guide for running local and paid AI models using `ollama`, `openrouter` and `opencode` on a fedora laptop with 8GB of RAM.

## table of contents

#### part 1 - hardware
- [hardware overview](#hardware-overview)

#### part 2 - strategy and component selection
- [strategy](#strategy)

#### part 3 - software setup
- [ollama](#ollama)
- [openrouter](#openrouter)
- [opencode](#opencode)

#### part 4 - performance improvements
- [hardware acceleration setup](#hardware-acceleration-setup)
- [opencode skills](#opencode-skills)

#### part 5 - conclusion
- [issues](#issues)

<br/>

## hardware overview
the target machine is severely memory-constrained. the **8GB RAM** drove every architectural decision, this setup tries to compensate with aggressive swap management and integrated gpu offloading.

| component | configuration |
|---|---|
| system model | ASUS Zenbook 14 UX3405MA_Q415MA |
| processor | Intel Core Ultra (Meteor Lake), Turbo Boost disabled |
| graphics | Intel Arc Integrated GPU (iGPU) |
| memory | 8GB onboard RAM |
| swap | 24 GB total. ZRAM priority: 100, disk swapfile priority: -1 |
| os | fedora gnome 44 |

<br/>

## strategy
due to the 8GB RAM limitation large models like 7b-8b and heavy electron/ docker-based harnesses were excluded. the chosen stack prioritized light-weight native binaries and sub 4b models.
- backend engine: ollama - linux native. chosen for bare-metal execution and vulkan gpu support.
- agent ahrness: opencode. chosen as a lightweight cli

<br/>

#### models in use

- qwen3-4b-agent:latest                                 
- llama3.2-agent:latest                            
- qwen3-1.7b-agent:latest                       
- qwen3:1.7b                                    
- qwen3:4b-instruct                                   
- qwen2.5-coder-agent:latest               
- qwen2.5-coder-lite:latest                            
- huihui_ai/llama3.2-abliterate:3b                      
- richardyoung/qwen2.5-3b-instruct-abliterated:latest   
- llama3.2:3b                                          
- qwen2.5-coder:3b      

<br/>


<br/>

## ollama

#### disable daemon at boot
to stop the AI engine form idling in the background and starving the fedora desktop of memory, the ollama systemd service was disabled

```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
```

<br/>

#### immediate RAM reclamation
by default, ollama holds models in memory for 5 minutes after a prompt. to force immediate eviction from RAM upon task completion, the systemd service was overidden.
```bash
# Added to /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_KEEP_ALIVE=0"
```

<br/>

#### initial context window capping
standard `qwen 2.5` model defaults to a **32,0000 token context window**, which consumes excessive RAM dynamically. an initial attempt was made to hard-cap this memory to **8,192 tokens** by compiling a custom `Modelfile`.

```bash
FROM qwen2.5-coder:3b
PARAMETER num_ctx 8192
```
> the resulting local model was tagged as `qwen2.5-coder-lite`

<br/>

## opencode
the initial configuration for the opencode cli harnessed the ollama backend via an OpenAI compatible translation layer. the configuration file at ~/.config/opencode/opencode.json was updated to achieve this.
```bash
{
  "model": "ollama/qwen2.5-coder-lite",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "baseURL": "http://127.0.0.1:11434/v1",
      "models": {
        "qwen2.5-coder-lite": {
          "name": "qwen2.5-coder-lite",
          "tools": true
        }
      }
    }
  }
}
```

<br/>

## hardware acceleration setup
to offload matrix mathematics from the cpu, fedora's native mesa **vulkan drivers** were installed to hook into the intel core ultra's `meteor lake` architecture.
```bash
sudo dnf install mesa-vulkan-drivers vulkan-tools intel-opencl intel-level-zero
```
> success can be verified by using vulkaninfo utility

<br/>

because ollama safely ignores integrated GPUs by default to prevent screen tearing, an explicit override flag was added to the **systemd** service
```bash
Environment="OLLAMA_IGPU_ENABLE=true"
```

<br/>

## issues
despite hardware optimization functioning correctly the software stack repeatedly failed to execute agentic tasks. `inetl_gpu_top` shows 72% render/ 3D usage. following are the reasons for failer.

- context starvation: the model output raw JSON instead of executing file scans.

diagnosis: opencode requires a massive 64k context windows just to pass its internal tool instructions. the 8k RAM-protective cap truncated these instructions, causing the model to fail.

<br/>

- GPU math callapse (NaN errors): to accommodate the 64k context in 8GB of RAM experimental memory compression flags like `OLLAMA_FLASH_ATTENTION=1` and `OLLAMA_KV_CACHE_TYPE=q8_0` are added. results are total blank output.

diagnosis: vulkan compute drivers failed to process the heavy quantized key-value cache, result is **Not A Number (NaN)** output.

<br/>

- API translation bug: the model continues to print raw JSON tool requests instead of executing them.

diagnosis: `@ai-sdk/openai-comatible` wrapper in the config file corrupted opencode native tool-calling parsers.

<br/>

- unwanted cloud fallback: after stripping the config to force native local connections, opencode entirely bypasses the local models. it automatically fell back to a free cloud model - "big pickle by opencode zen"

diagnosis: if the ollama background service is not running at the exact moment opencode launches, the harness marks the local provider as **unavailable** and defaults to the cloud.


