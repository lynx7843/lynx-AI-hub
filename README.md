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
### disable daemon at boot
to stop the AI engine form idling in the background and starving the fedora desktop of memory, the ollama systemd service was disabled

```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
```



