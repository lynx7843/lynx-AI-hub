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
- [os configuration](#os-conguration)
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
standard `qwen 2.5` model defaults to a **32,0000 token context window**, which consumes excessive RAM dynamically. AN initial attempt was made to hard-cap this memory to **8,192 tokens** by compiling a custom `Modelfile`.

```bash
FROM qwen2.5-coder:3b
PARAMETER num_ctx 8192
```
> the resulting local model was tagged as `qwen2.5-coder-lite`






