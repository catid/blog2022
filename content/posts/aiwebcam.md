+++
title = "Why WebRTC is our AI Future"
date = "2023-11-20"
cover = "posts/aiwebcam/logo256.png"
description = "Mean, multi-modal and in your face"
+++

## WebRTC <3 AI

If you've seen "The Artifice Girl" on Amazon Prime Video you know where this is going, but either way we're going to get into the technical details as usual.

One of the promises of multi-modal AI like GPT-4-Vision is that you'll be able to have a real conversation with a computer in real-time.  The ChatGPT app on your phone can have real-time conversations with you using the OpenAI web API, which is a cool demo, but it doesn't produce text-mode output or accept vision-mode input so it is much less useful than public domain projects right now.

Bringing all the modalities together is possible using a web browser communicating with a WebRTC backend, similar to a Zoom call.  The webpage can also host the other modalities supported by AI models such as text-to-image, text-to-text, and so on.  But to really deliver on the promise of real-time conversation, WebRTC is ready to go today: The latency between speaking and getting a response back is about the same as a normal conversation!

Here's an example of live multi-modal conversation with an AI: https://www.youtube.com/watch?v=G_L8t3EQMcs

The full code for the examples below is available here: https://github.com/catid/aiwebcam2

Note this code is in Python and plain HTML/Javascript so it's more useful for people without a learning curve, but the same ideas presented here work in all programming languages for production uses.

This post is lovingly hand-crafted by a real human and may contain human flaws.


## Elements of an AI Conversation

So what do we need to do to replicate a Zoom call with an AI?  Let's see where we can cut down latency.

First we need to listen to the user's microphone and decide what parts of the audio from the microphone is their speech.  We then need to convert the speech audio to text.  Then the text needs to be interpreted in context of the rest of the conversation by the AI and response text streamed back from the OpenAI servers.  While the text is streaming back, we need to start converting the text into an audio stream.  Then while the audio is streaming from the OpenAI servers we need to start playing it back from the user's speaker.

Note that the speech to text requires full context of the audio sample, so we need to wait for the ASR operation to complete before passing text to the OpenAI server.  The OpenAI server needs full context from our text, but it does stream output as it is generated.  Once we get a complete sentence, something we can easily parse with ". " or "? " or newline delimiters, we can start generating audio using the OpenAI TTS API right away.

But how do we access the user's microphone and play back audio with low latency?  WebRTC solves all of this!  Regardless of what language the WebRTC server-side is implemented in, you'll get the lowest latency possible using this technology, and it even works right from a browser on the client side.  Microphone samples stream as fast as possible and are received in (usually) 20 msec Opus audio clips.  Audio samples can be delivered back to the browser in 20 msec Opus audio clips as soon as they are ready.


## WebRTC in Python with OpenAI API

With the pace of AI progress, any projects done in this space are probably written in Python.  But can we do low-latency WebRTC using Python?

Absolutely.  Using the `aiortc` Python library, you can host a WebRTC server from your Python script.

The browser require HTTPS to use WebRTC, so you'll need to create an SSL key on the console:

```bash
openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 3650
```

Then in your Python script, add an HTTPS server hosting a static page:

```python
import aiohttp.web

app = aiohttp.web.Application()
sio.attach(app)

async def handle_index(request):
    if request.path == "/":
        with open("./static/index.html") as f:
            text = f.read()
            return aiohttp.web.Response(text=text, content_type="text/html")

app.router.add_route("GET", "/", handle_index)  # Serve index.html for the root URL path
app.router.add_route("GET", "/index.html", handle_index)  # Serve index.html for the /index.html URL path
app.router.add_static("/", "./static")  # Serve files from the ./static/ directory using the root URL path

ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ssl_context.load_cert_chain(args.cert_file, args.key_file)

aiohttp.web.run_app(app, host=args.host, port=args.port, ssl_context=ssl_context)
```

The static page needs video and audio elements to mirror the webcam/desktop share and to play back the TTS audio:

```html
<div id="video-container">
    <video id="video" muted autoplay playsinline>
        Your browser does not support the video element.
    </video>
    <audio id="audio" autoplay playsinline>
        Your browser does not support the audio element.
    </audio>
</div>
```

Using socket.io, the details of which I'll gloss over since this is extremely common, you can handle multiple sessions and reconnection logic when the user refreshes or the server restarts.

When the browser establishes a socket.io connection, the backend can create a WebRTC connection object.  To establish a WebRTC connection, the browser initiates with:

```javascript
const socket = io();
const configuration = { iceServers: [{ urls: "stun:stun.l.google.com:19302" }] };
let peerConnection;

socket.on("connect", () => {
    // Connected to signaling server.  Negotiating video...
    peerConnection = new RTCPeerConnection(configuration);

    peerConnection.onconnectionstatechange = onConnectionStateChange; // See below
    peerConnection.ontrack = onTrack; // See below

    // Add media stream to the peer connection
    const stream = await getWebcamMediaStream(); // See below
    if (stream) {
        stream.getTracks().forEach((track) => peerConnection.addTrack(track, stream));

        const video = document.getElementById("video");
        video.srcObject = stream;
    }

    // This is where we send the WebRTC offer to the backend Python script.
    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    await socket.emit("my_webrtc_message", { sdp: offer });
});
```

This is how to get the media stream for the user's webcam.  It usually pops up a dialog that the user can accept/reject:

```javascript
async function getWebcamMediaStream() {
    const constraints = {
        audio: true,
        video: {
            width: { ideal: 640, max: 1920 },
            height: { ideal: 480, max: 1080 },
        },
    };
    return await navigator.mediaDevices.getUserMedia(constraints);
}
```

And this is how to handle the WebRTC events in the browser:

```javascript
async function onTrack(event) {
    const video = document.getElementById("video");
    const audio = document.getElementById("audio");
    if (event.track.kind === 'video' && !video.srcObject && event.streams[0]) {
        video.srcObject = event.streams[0];
    }
    else if (event.track.kind === 'audio' && !audio.srcObject && event.streams[0]) {
        audio.srcObject = event.streams[0];
    }
}

async function onConnectionStateChange(event) {
    if (peerConnection.connectionState === "disconnected" ||
        peerConnection.connectionState === "failed") {
        // WebRTC disconnected. Reconnecting...
        await peerConnection.close();
        await startWebRTC();
    }
}
```

You can also add a button that shares an application from the user's desktop instead of the webcam:

```javascript
// Function to replace the video track in the peer connection
async function replaceVideoTrack(stream) {
    // Get the new video track from the stream
    const videoTrack = stream.getVideoTracks()[0];
  
    // Find the sender that is transmitting the current video track
    const sender = peerConnection.getSenders().find(s => s.track && s.track.kind === 'video');
  
    if (sender) {
      // Replace the track with the new one
      sender.replaceTrack(videoTrack);
    } else {
      // If there's no video sender, add a new one
      peerConnection.addTrack(videoTrack, stream);
    }
}

document.getElementById('desktopButton').addEventListener('click', function() {
    if (navigator.mediaDevices && navigator.mediaDevices.getDisplayMedia) {
        // Request desktop sharing
        navigator.mediaDevices.getDisplayMedia({ video: true })
            .then(function(stream) {
                // You now have a stream of the desktop
                // Here you would typically attach it to a video element or handle it further
                console.log('Got desktop stream:', stream);

                replaceVideoTrack(stream)

                const video = document.getElementById("video");
                video.srcObject = stream;
            })
            .catch(function(error) {
                // Handle errors, for example, the user denied the request
                console.error('Error getting display media:', error);
            });
    } else {
        alert('getDisplayMedia API is not supported by this browser.');
    }
});
```

Now on the Python backend, you can handle the WebRTC session start message:

```python
@sio.event
async def my_webrtc_message(sid, data):
    session = sessions.find_session_by_sid(sid)
    if session:
        logger.info(f"Received WebRTC start message from {sid}: {data}")

        browser_sdp = data["sdp"]

        local_description = await session.rtc_peer_connection.create_session_answer(browser_sdp) # see below

        if local_description:
            await sio.emit("session_response", {"sdp": local_description.sdp, "type": local_description.type}, room=sid)

async def create_session_answer(self, browser_sdp):
    description = RTCSessionDescription(sdp=browser_sdp["sdp"], type=browser_sdp["type"])

    await self.pc.setRemoteDescription(description)

    self.pc.addTrack(self.opus_track) # Add TTS audio track!  See below!

    await self.pc.setLocalDescription(await self.pc.createAnswer())

    return self.pc.localDescription

```

Finally back in Javascript we need to handle the response to complete the handshake:

```javascript
socket.on("session_response", async (data) => {
    const description = new RTCSessionDescription(data);
    await peerConnection.setRemoteDescription(description);
});
```

Now you have a WebRTC connection!  How do we get audio from the user's microphone?

In Python you handle the "track" event on the RTCPeerConnection object, which is set up after you create it:

```python
self.pc = RTCPeerConnection()

@self.pc.on("track")
def on_track(track):
    if track.kind == "audio":
        if not self.processing_audio:
            self.processing_audio = True
            asyncio.create_task(self.process_audio_track(track)) # see below
        pass

    async def process_audio_track(self, track):
        while True:
            try:
                frame = await track.recv()

                # This is where you can decide to start recording the audio samples.
                # These are decoded PCM audio samples in pyAV Frame objects.
                if self.recording:
                    self.audio_sample_rate = frame.sample_rate
                    self.audio_channels = len(frame.layout.channels)
                    self.audio_frames.append(frame.to_ndarray())
            except MediaStreamError:
                # This exception is raised when the track ends
                break
```

So now we're able to record audio from the user.

For my project, instead of trying to automatically figure out when the user wants to speak to the AI, which I have yet to see done well, I just allow the user to click the page or hold spacebar to start recording an audio snippet.  This is simple and works 100% of the time.  Probably someone needs to release an AI model that just specializes in listening to a user and knowing when they're trying to talk to the AI, but that does not exist yet as far as I know, and is probably only useful for kiosks or places that have limited input.

To record video from the user we need to subclass the VideoStreamTrack object from aiortc:

```python
class VideoReceiver(VideoStreamTrack):
    kind = "video"

    def __init__(self, track):
        super().__init__()  # Initialize the MediaStreamTrack
        self.track = track
        self.recording = False
        self.recorded_frame = None

    def startRecording(self):
        self.recording = True
        self.recorded_frame = None

    def endRecording(self):
        self.recording = False
        image = self.recorded_frame
        self.recorded_frame = None
        return image

    async def recv(self):
        frame = await self.track.recv()

        # Process the frame (e.g., save to a file, play audio, etc.)
        if self.recording:
            if not self.recorded_frame:
                self.recorded_frame = frame

        return frame
```

This implementation allows other code in Python to set the `VideoReceiver.recording = true` to get it to fill in `self.recorded_frame` with a frame.  This is a PyAV Frame object, which holds a decoded image.  Since it's a webcam, the format of the image is YUV420 (same colorspace as JPEG), but we still need to convert it to a format we can share with GPT-4-Vision.  Read below for how to do that.

Skipping over the OpenAI parts for now, let's cover how to get audio back out to the user's speaker.

As you saw a while ago you can add an `opus_track` track to the `pc` object to start an audio stream going back to the user.  To implement that we need to subclass `MediaStreamTrack``:

```python
self.opus_track = CustomAudioStream()

class CustomAudioStream(MediaStreamTrack):
    kind = "audio"

    def __init__(self):
        super().__init__()  # don't forget this!

        self.tts = TTSServiceRunner() # See below

        self.stream_time = None

    async def close(self):
        super().stop()
        self.tts.close()

    async def recv(self):
        packet, duration = self.tts.poll_packet() # See below

        if self.stream_time is None:
            self.stream_time = time.time()

        wait = self.stream_time - time.time()
        await asyncio.sleep(wait)

        self.stream_time += duration
        return packet
```

It uses the TTS packet durations to keep pace with the stream and release packets as they need to be delivered to the user.  Note that the release time does not need to be perfect - Actual playback buffering is handled by the WebRTC code built into the browser and is based on the timestamps in the packets as you'll see later.

This uses a TTS object we'll define later to get audio packets.  It's important that the tts.poll_packet() function never blocks.  If there is no data, it needs to return an Opus packet that represents 20 milliseconds of silence because otherwise the main Python thread will block and everything will hang waiting for the packet to be ready.

Okay so now we have the ability to read audio/video from the user and to generate speech audio.


## Best Quality Real-time Speech-to-Text

After we have the PCM audio from the user's microphone, we can process it using the open-source Whisper 3 model.  Since OpenAI values user privacy they have open-sourced the Whisper ASR models so that you do not need to share any audio from your microphone with OpenAI.  It's also faster to host this locally and reduces latency overall.  Whisper 3 is about twice as fast as Whisper 2 and can be made 25% faster using the BetterTransformers package as demonstrated below.  In the future OpenAI will distill a small version of the model that can shortcut the big model and will make it about 2x faster again, so it is already blazing fast and super-high quality.

Since the ASR operation is pretty heavy and we don't want to block the main Python thread, we run the ASR processing in a background process and use multiprocessing.Queue objects to interact with the model.  The details are in the repo: https://github.com/catid/aiwebcam2/blob/main/service_asr.py

Here's how to set up the basic Whisper 3 pipeline:

```python
device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

model_id = "openai/whisper-large-v3"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True
)
model = model.to_bettertransformer()
model.to(device)

processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model,
    tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    max_new_tokens=128,
    chunk_length_s=30,
    batch_size=16,
    return_timestamps=True,
    torch_dtype=torch_dtype,
    device=device,
)

...

result = pipe(pcm_data)

text = result["text"]
```

I believe the model now will internally resample the audio to 16 kHz, but we can do that ourselves to improve quality.  We can also remove silences in the input audio to reduce processing time.  Furthermore sometimes the user accidentally activates the microphone briefly and records silence so we should avoid passing dead air to the Whisper 3 model since it will try to interpret the noise as speech.

Since the input from the browser is actually an array of short PCM samples, we have to concatenate them all, make them mono if they are stereo, and reformat the data to work with the pipeline:

```python
self.whisper_sample_rate = 16000
self.fast_resampler = samplerate.Resampler('sinc_fastest', channels=1)
self.best_resampler = samplerate.Resampler('sinc_best', channels=1)

def normalize_audio(self, pcm_data_array, channels, input_sample_rate):
    normalized_array = []

    for pcm_data in pcm_data_array:
        # Squeeze out extra dimensions:
        if pcm_data.ndim > 1 and (pcm_data.shape[0] == 1 or pcm_data.shape[1] == 1):
            pcm_data = np.squeeze(pcm_data)

        # Check if audio is stereo (2 channels). If so, convert it to mono.
        if pcm_data.ndim > 1 and pcm_data.shape[1] == 2:
            pcm_data = np.mean(pcm_data, axis=1)
        elif pcm_data.ndim == 1 and channels == 2:
            # Left channel
            pcm_data = pcm_data[::2]

        if np.issubdtype(pcm_data.dtype, np.floating):
            # Convert to int16, PyDub doesn't work with float
            pcm_data = (pcm_data * np.iinfo(np.int16).max).astype(np.int16)

        normalized_array.append(pcm_data)

    pcm_data = np.concatenate(normalized_array)

    # Create an audio segment from the data
    sound = AudioSegment(
        pcm_data.tobytes(), 
        frame_rate=input_sample_rate,
        sample_width=pcm_data.dtype.itemsize, 
        channels=1
    )

    # Remove silence from audio clip
    chunks = split_on_silence(sound, 
        min_silence_len=500, # milliseconds
        silence_thresh=-30, # dB
        keep_silence = 300, # milliseconds
        seek_step=20 # Speed up by seeking this many ms at a time
    )

    # Ignore empty audio
    if len(chunks) <= 0:
        return None

    non_silent_audio = sum(chunks)

    byte_string = non_silent_audio.raw_data
    int_audio = np.frombuffer(byte_string, dtype=np.int16)
    float_audio = int_audio.astype(np.float32) / np.iinfo(np.int16).max

    # Use a faster resampler for longer audio
    if len(float_audio) > 100000:
        resampler = self.fast_resampler
    else:
        resampler = self.best_resampler

    # Resample audio to the Whisper model's sample rate
    pcm_data = resampler.process(
        np.array(float_audio),
        self.whisper_sample_rate / input_sample_rate)

    # Ignore very short audio clips
    if len(pcm_data) / self.whisper_sample_rate < 0.5:
        return None

    return pcm_data
```

This cleanup step cuts out a lot of silence as the user is saying "um" or pausing while thinking.  It also cuts out audio where the user accidentally clicked the page and turned on their microphone by accident, which happens a lot.  I've tuned it so that these accidental clicks do not trigger any response from the AI.


## Converting Webcam Frames to Base64 JPEG

We saw earlier how to get Webcam Frames from `aiortc`.  The PyAV library stores decoded YUV420 images in a 2D ndarray, where the Y, U, and V frames are concatenated:

```python
image = recorded_frame.to_ndarray(format='yuv420p') # This may convert to YUV if it is some other format

width = image.shape[1]
yuv_h = image.shape[0]
height = round(yuv_h * 2 / 3)

image = convert_yuv420_to_ycbcr(image, width, height) # See below

img = Image.fromarray(image, 'YCbCr')

buffer = io.BytesIO()
img.save(buffer, format='JPEG', quality=95)

buffer.seek(0)

# Result:
base64_image = base64.b64encode(buffer.read()).decode('utf-8')


def convert_yuv420_to_ycbcr(data, w, h):
    # Extract Y, U, and V channels
    Y = data[:h]
    U = data[h:(h + h//4)].reshape((h // 2, w // 2))
    V = data[(h + h//4):].reshape((h // 2, w // 2))

    # Upsample U and V channels
    U_upsampled = U.repeat(2, axis=0).repeat(2, axis=1)
    V_upsampled = V.repeat(2, axis=0).repeat(2, axis=1)

    # Stack the channels
    ycbcr_image = np.stack([Y, U_upsampled, V_upsampled], axis=-1)

    return ycbcr_image
```

This is how you can convert the webcam image into a base64 coded JPEG that can be shared with the OpenAI conversation API for GPT-4-Vision.  The whole operation takes about 1 millisecond so you don't need to run this on a background thread.  In my project I only do this for one image not for every image in the stream.


## Using GPT-4-Vision with Streaming Output

Now that we have the text the user spoke, and the webcam images as base64 JPEG, we can query OpenAI for conversation completion.

You'll want to keep track of the conversation history and convert it to the JSON format that OpenAI GPT-4-Vision supports:

```python
prompt_messages = []

prompt_messages.append({
    "role": line.sender,
    "content": [
        {
            "type": "text",
            "text": "user" # also supports "assistant" but no other values for now
        },
        {
            "type": "image_url",
            "image_url": {
                "url": f"data:image/jpeg;base64,{base64_image}", # See above for how to generate this
                "detail": "high" # "low" and "auto" also work
            }
        }
    ],
})

system_message = {
    "role": "system",
    "content": [
        {
            "type": "text",
            "text": "You are a helpful AI assistant that can see the user."
        },
    ],
}
prompt_messages.insert(0, system_message)
```

Since the OpenAI completion API blocks the Python main thread we run it in a background process and access it via multiprocessing.Queue objects.  Details are in the repo: https://github.com/catid/aiwebcam2/blob/main/service_llm.py

We want to use the streaming mode so that we can start the text-to-speech as soon as possible and also write the output from the model to the web page as fast as we can.  To do that, just specify `stream=True` and wait for the responses:

```python

import api_key # Modify the api_key.py file to specify your OpenAI key, which you generate here: https://platform.openai.com/api-keys

import openai
client = openai.OpenAI(api_key=api_key.api_key)

response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    messages=prompt_messages,
    temperature=0.5,
    max_tokens=4096,
    n=1,
    stream=True # Streaming output!
)

fulltext = ""

for completion in response:
    text = completion.choices[0].delta.content
    if text:
        fulltext += text

        self.response_queue.put(fulltext)

self.response_queue.put(fulltext)
```

The way I'm handling this is building the string as the chunks come in, and then processing the growing string downstream.

To engage the text-to-speech faster, I'm seeking for the end of the last sentence in the received text and speaking as soon as we have a complete sentence to say:

```python
class TextParser:
    def __init__(self):
        self.next_spoken_offset = 0
        self.found_newline = False

    async def on_message_part(self, message, pc):
        if self.found_newline:
            return

        if not pc:
            return
        tts = pc.opus_track.tts
        if not tts:
            return

        message = message[self.next_spoken_offset:]

        last_index = message.find("\n")
        if last_index != -1:
            # Truncate to before the newline and speak it all.
            ready_text = message[:last_index]
            self.found_newline = True
        else:
            # Break at the last sentence ending
            substrings = [". ", ": ", "! ", "? "]
            last_indices = [message.rfind(substring) for substring in substrings]
            last_index = max(last_indices)
            if last_index == -1:
                return
            ready_text = message[:last_index+2]
            self.next_spoken_offset += last_index + 2

        ready_text = ready_text.strip()

        if ready_text:
            await tts.Speak(ready_text) # Speak it!

    async def on_message(self, message, pc):
        if not pc:
            return
        tts = pc.opus_track.tts
        if not tts:
            return

        if not self.found_newline:
            def truncate_at_newline(s):
                return s.split('\n', 1)[0]

            message = truncate_at_newline(message)
            message = message[self.next_spoken_offset:].strip()

            if message:
                await tts.Speak(message) # Speak it!

        self.next_spoken_offset = 0
        self.found_newline = False
```

Now as soon as we can the AI will start producing speech to reduce latency further!


## Using OpenAI Text-to-Speech (TTS)

OpenAI recently also released a new super-fast TTS service with realistic voices.  It has a slow HD mode good for generating audio-books.  ( See my project here: https://github.com/catid/AutoAudiobook )  And it also has a fast real-time speech mode for conversational AI.  It's much slower than open-source TTS solutions that are free to use and you can host yourself, but the quality is so much higher it's just much more pleasant to listen to since it sounds much more emotional.  You can compare the YouTube video demo speech from the top of this blog with my previous blog sharing the best open-source fast TTS solution: https://catid.io/posts/tts/  In my impression, the OpenAI TTS could be mistaken for a real human, which makes it worth using, and it does a much better job pronouncing technical jargon.

The TTS API supports HTTP streaming output in ~1 second chunks, which is sigificantly faster than waiting for the full audio clip to arrive.  Note that it generates audio faster than real-time so in practice we get the first usable Opus audio clip within about 700 milliseconds for an average real query.  We can ask for Opus formatted audio (in an OGG format), which is great because WebRTC supports Opus audio for compression.  Since we don't need to transcode the audio we can save some time!

Unfortunately Python does not have good tools for working with Ogg Opus containers.  I had to write my own parser for this format, which is available here: https://github.com/catid/aiwebcam2/blob/main/service_tts.py

Here's how to use the OpenAI TTS API and stream the audio chunks:

```python
def streamed_audio(input_text, callback, voice='alloy', model='tts-1', speed=1.0):
    #t0 = time.time()

    # OpenAI API endpoint and parameters
    url = "https://api.openai.com/v1/audio/speech"
    headers = {
        "Authorization": f"Bearer {api_key.api_key}",
    }

    data = {
        "model": model,
        "input": input_text,
        "speed": speed,
        "voice": voice,
        "response_format": "opus",
    }

    with requests.post(url, headers=headers, json=data, stream=True) as response:
        if response.status_code != 200:
            return False

        buffer = b''
        next_page_index = 0

        def queue_results(callback, page_index, parsed, buffer_complete):
            if not parsed.info:
                return False

            chunks = parsed.get_page(page_index, buffer_complete)

            if not chunks:
                return False

            channel_count = parsed.info["channel_count"]
            sample_rate = parsed.info["input_sample_rate"]

            for chunk in chunks:
                callback(channel_count, sample_rate, chunk)

            return True

        for chunk in response.iter_content(chunk_size=16384):
            buffer += chunk

            parsed = OpusFileSplitter(buffer)
            while queue_results(callback, next_page_index, parsed, buffer_complete=False):
                next_page_index += 1

        parsed = OpusFileSplitter(buffer)
        while queue_results(callback, next_page_index, parsed, buffer_complete=True):
            next_page_index += 1

    return True
```

Using large chunk sizes does not slow down the HTTP reads, so you might as well.  The HTTP chunks are about 4-8KB on average since Opus codec is a variable-length audio codec.  Some of the code above is harder to read because it's using my parser in the full repo, but basically as soon as Opus audio clips become available in the stream, they're loaded into PyAV Packets and shipped to the browser via WebRTC.  I'll discuss that in the next section.

The one problem that I found with the OpenAI TTS endpoint is that the speed parameter needs to be 1.0 since the output sounds bad at faster speeds.  You might as well use lower quality and free TTS for fast-speaking audio, so OpenAI should probably fix this to remain competitive.


## Streaming Ogg Opus via WebRTC

The basics of Ogg Opus are that you get "pages" that contain short audio clips coded with Opus.  Each page has a number of little clips, each with a duration.  Unfortunately the duration seems to only be available if you decode the clip.  For OpenAI all the clips are 20 milliseconds long, but in case they change that I go ahead and decode them anyway to be sure.

A silent clip is just three bytes regardless of sample rate etc: `bytes.fromhex('f8 ff fe')` so you don't need to spin up an Opus encoder to produce 20 milliseconds of silence.

As soon as my parser above produces an Opus packet, we queue it up for delivery to the browser.  I removed the queuing details from the TTSServiceRunner so you see how it's grabbing the Opus clips from the background process, and either assigning a presentation timestamp (PTS) and display timestamp (DTS) to a silent packet or the dequeued packet:

```python
# RTP timebase needs to be 48kHz: https://datatracker.ietf.org/doc/rfc7587/
time_base = 48000
time_base_fraction = fractions.Fraction(1, time_base)

class TTSServiceRunner:
    def __init__(self):
        self.next_pts = 0

    def generate_silence_packet(self, duration_seconds):
        chunk = bytes.fromhex('f8 ff fe')

        packet = av.packet.Packet(chunk)
        packet.pts = self.next_pts
        packet.dts = self.next_pts
        packet.time_base = time_base_fraction

        pts_count = round(duration_seconds * time_base)
        self.next_pts += pts_count

        #logger.info(f"silence pts_count = {pts_count}")

        return packet

    # Grab either the next TTS Opus packet to play back,
    # or a silence packet if no data is available.
    def poll_packet(self):
        try:
            duration, pts_count, chunk = self.response_queue.get_nowait()

            packet = av.packet.Packet(chunk)
            packet.pts = self.next_pts
            packet.dts = self.next_pts
            packet.time_base = time_base_fraction

            self.next_pts += pts_count

            return packet, duration

        except:
            pass # Ignore Empty exception from get.nowait()

        silence_duration = 0.02
        return self.generate_silence_packet(silence_duration), silence_duration
```

The poll_packet() function is the one we showed earlier and finally completes the CustomAudioStream class implementation to deliver TTS audio to the browser.


## Discussion

To recap, we've shown how to get as low latency as possible with the GPT-4-Vision, TTS, and Whisper 3 models right from Python in a way that's easy to hack on and use for one-off projects.

I hope that others find the code useful and a good starting point for building their own projects based on OpenAI tools: https://github.com/catid/aiwebcam2

Future work:

* Add a cancel button so the AI does not talk over you.
* Improve the HTML render frame and UI in general to be more usable with resizeable frames and copy buttons for generated code.
* Have a button to switch between desktop apps and user's webcam.
* Support for other browsers and iPhone.
* Use Unreal engine to generate a real-time lip-synced avatar for the AI running on the server.
* Listen to audio and decide when to respond more intelligently.
* Integrate with a Zoom client to allow the AI to join teleconferences and reply.


## Multi-modal is awesome:

Why is Text-to-Speech (TTS) awesome?  Parsing a huge wall of text is a challenge for most people.  Allowing the AI to speak increases the human interaction surface area of the AI, allowing you to more easily absorb the information being provided.  Since the TTS rarely makes mistakes and never says "um", doesn't have any sponsors to flog, and doesn't need to pad run-time, it is a much faster way to get information than looking for YouTube videos on a topic for example.  It's important to be able to interrupt the speech output or make it speak faster to really have the best experience with this.  For disabled people or in kiosk usage and other cases where generated text is harder to consume, TTS is a game-changer.

Why is Automated Speech Recognition (ASR), speech-to-text, awesome?  Speaking is actually a faster way to do data entry than writing text, though can be less accurate.  Since GPT-4 and newer AIs are very forgiving with typos and can infer intent from context, you don't really need perfect text input so speech is unusually effective.  Since co-designing with an AI involves a lot of back-and-forth refinement, speaking your intent is a good way to speed up the interaction.

We should already be convinced that text-to-text large language models at the size of GPT-4 are amazingly useful for professional uses.  In the context of their vast training, you can get a huge leg-up in a new project over someone working from scratch.  They're fantastic at translating between modalities in general, such as from a description to code, or from code in one language to another.

Why is AI image generation awesome?  DALL-E 3 is currently the best AI art generator model, and it's free to use!  I've been using it to generate artwork for all sorts of things.  This basically makes Midjourney, SDXL, and prior efforts not worth investing the time to learn.  They are the first solution that gets hands and text and repeated patterns correct, and the researchers who figured out the trick actually open-sourced how they did it so that Midjourney and SDXL can eventually catch up.

Text-to-music is on the way...  We'll see what people end up using this for!
