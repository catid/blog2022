+++
title = "Can open-source LLMs detect bugs in C++ code?"
date = "2023-04-15"
cover = "posts/llm_bugs/llm_bugs.jpg"
description = "The answer may surprise you."
+++

# No

No, they cannot.

```
LLaMa 65B (4-bit GPTQ) model: 1 false alarms in 15 good examples.  Detects 0 of 13 bugs.
Baize 30B (8-bit) model: 0 false alarms in 15 good examples.  Detects 1 of 13 bugs.
Galpaca 30B (8-bit) model: 0 false alarms in 15 good examples.  Detects 1 of 13 bugs.
Koala 13B (8-bit) model: 0 false alarms in 15 good examples.  Detects 0 of 13 bugs.
Vicuna 13B (8-bit) model: 2 false alarms in 15 good examples.  Detects 1 of 13 bugs.
Vicuna 7B (FP16) model: 1 false alarms in 15 good examples.  Detects 0 of 13 bugs.

GPT 3.5: 0 false alarms in 15 good examples.  Detects 7 of 13 bugs.
GPT 4: 0 false alarms in 15 good examples.  Detects 13 of 13 bugs.
```

Reproduce my results here: https://github.com/catid/supercharger/tree/main/airate
