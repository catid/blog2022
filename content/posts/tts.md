+++
title = "Text-to-speech"
date = "2023-04-16"
cover = "posts/tts/everything.jpg"
description = "Offline, fast, good, and free"
+++

## Text-to-speech

Probably a lot of people are interested in talking to their LLMs via speech.  To get realistic human voices from a computer, you need to use text-to-speech (TTS).

I found one that is super simple to get going:

```bash
pip install TTS

tts --model_name tts_models/en/ljspeech/tacotron2-DDC_ph --vocoder_name vocoder_models/en/ljspeech/univnet --text "When I choose to see the good side of things, I'm not being naive. It is strategic and necessary. It's how I've learned to survive through everything.  I know you go through life with your fists held tight.  You see yourself as a fighter. Well, I see myself as one too. This is how I fight." --progress_bar True --use_cuda True
```

Here's the output:

{{< rawhtml >}}
<audio controls>
  <source src="https://catid.io/posts/tts/tts_output.wav" type="audio/wav">
  Your browser does not support the audio element.
</audio>
{{< /rawhtml >}}

This model will split up the input into sentences and then generate a wav file for each sentence.  So, a good idea to reduce latency from LLM output is to split the LLM output at sentence boundaries since they are easy to separate by scanning for periods followed by a space.

Then to reduce latency, the LLM can start speaking as soon as the first sentence is done being generated, and the next sentence can be generated while the first sentence is being spoken.