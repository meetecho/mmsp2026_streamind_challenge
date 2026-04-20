# The STREAMIND challenge

Welcome to the STREAMIND Grand Challenge!

The objective of the challenge is to design and implement a complete streaming
pipeline, bridging the gap between live audio reception and structured language
model inference. Participants will build their solutions on top of
[Juturna](https://github.com/meetecho/juturna), an open-source Python framework
developed by [Meetecho](https://www.meetecho.com/en/) for real-time AI data
pipeline prototyping.

## Important dates

- Registration deadline: April 3, 2026
- Paper submission deadline: June 19, 2026
- Acceptance notification: July 17, 2026

To register, submit a form [here](https://forms.gle/jED7zCjWRCVwg7Bp9).

:information_source: **Registrations are now closed!**

## Challenge overview

The core task of this challenge is the implementation of a pipeline that
consumes live audio streams as inputs, and produces textual summaries and
keywords as outputs.

Roughly speaking, such pipeline can be organized around five main processing
steps:

1. Audio reception via WebRTC/RTP: a live audio stream is delivered as a
   Opus-encoded mono RTP stream on a dedicated port.
2. Incremental ASR transcription: the incoming audio stream is incrementally
   transcribed by an ASR node operating on short, consecutive and partially
   overlapping audio chunks of a set temporal length.
3. Novel chunk extraction: novel transcript text is isolated while content from
   previous audio chunks is discarded.
4. Window aggregation: transcription chunks are accumulated into context
   windows of 300 seconds.
5. Summarization: aggregated windows are processed to produce structured
   textual outputs.
6. Transmission: summary objects are stored locally on the filesystem and
   transmitted to a destination endpoint through `POST` requests.

![Overview](/images/challenge_overview.png)

## Datasets

We include in this repository 2 reference datasets that can be used to design
and test the submission pipelines:

1. The `rev16` dataset is a public collection of 30 podcast episodes, decorated
with their transcriptions. Here we provide the
[whisper subset](https://huggingface.co/datasets/distil-whisper/rev16/tree/main/whisper_subset),
used in the Whisper original paper for the long-form transcription evaluation.
2. The `ietf` dataset is a collection of 8 live sessions recorded during the
latest IETF conference, IETF125.

Both datasets are offered as collections of mono-channel opus files, sampled at
16000 Hz.

Furthermore, the implemented pipelines can be tested and validated against any
English speech audio files long enough to allow progressive summaries to be
generated. It is recommended to use audio content of at least 30 minutes, which
will result in 6 summary chunks.

All submissions will ultimately be evaluated against a small held-out dataset
similar to the ones included in this repository.

## Pipeline structure

A simple starting point for this challenge is a simple transcription pipeline
that consumes a remote RTP audio stream and transcribes it using Whisper, then
sends the transcribed chunks to a mongo endpoint. Such pipeline can be
configured with the following `JSON` file:

```json
{
  "version": "1.0",
  "plugins": [
    "./plugins"
  ],
  "pipeline": {
    "name": "pipe",
    "id": "test_pipeline",
    "folder": "./pipe",
    "telemetry": "telemetry.csv",
    "nodes": [
      {
        "name": "0_source",
        "type": "source",
        "mark": "audio_rtp",
        "configuration": {
          "rec_host": "0.0.0.0",
          "rec_port": 23450,
          "audio_rate": 16000,
          "block_size": 3,
          "channels": 1,
          "payload_type": 0,
          "encoding_clock_chan": "PCMU/8000"
        }
      },
      {
        "name": "1_vad",
        "type": "proc",
        "mark": "vad_silero",
        "configuration": {
          "rate": 16000,
          "keep": 1
        }
      },
      {
        "name": "2_trx",
        "type": "proc",
        "mark": "transcriber_whispy",
        "configuration": {
          "model_name": "medium",
          "language": "en",
          "task": "transcribe",
          "buffer_size": 5
        }
      },
      {
        "name": "3_mongo",
        "type": "sink",
        "mark": "notifier_mongo",
        "configuration": {
          "endpoint": "mongodb://your.endpoint.address:27017",
          "database": "target_database",
          "collection": "target_collection",
          "timeout": 2
        }
      }
    ],
    "links": [
      {
        "from": "0_source",
        "to": "1_vad"
      },
      {
        "from": "1_vad",
        "to": "2_trx"
      },
      {
        "from": "2_trx",
        "to": "3_mongo"
      }
    ]
  }
}
```

Once this pipeline is started, `ffmpeg` can be used to simulate a remote audio
source, using a target audio file:

```bash
$ ffmpeg \
      -readrate 1 \
      -i audio.opus \
      -f rtp \
      -ac 1 \
      -acodec pcm_mulaw \
      -ar 8000 \
      -safe 0 \
      -sdp_file _session.sdp \
      rtp://127.0.0.1:23450
```

## Submission output

Submissions should contain:

- the code for all the implemented custom components,
- the pipeline configuration file,
- the progressive summary objects, as described in the previous section,
- a `Dockerfile` that can be used to run the pipeline against a target audio
stream,
- a short markdown file containing with a brief explanation of the used
approach.

## Evaluation metrics

All submitted chunks for an audio source will be assigned a composite score.

Chunk $c_i$, produced in a time $l_i$ and associated with the reference summary
$ref_i$, contains the generated summary $gen_i$ and the keyword list
$[k_1, k_2, k_3]_i$. Its summary score $C_i$ is computed as sum of three
distinct components:

**Base score (max. 25 points)**: this is a composite metric metric extracted
using a LLM acting as a judge. The summary will be evaluated against 5 different
features, getting for each one a score in the Likert scale. The sum of these
scores represents the base score $B_i$.

- Factual consistency: does the summary only contain facts that are present in
the source document?
- Relevance: are all the important facts in the source document present in the
summary?
- Coherence: is the summary well-structured and well-organised? does it
represent a coherent body of information about a particular topic?
- Fluency: is the summary well-written and free of formatting and grammatical
errors or issues?
- Conciseness: does the summary convey its information while keeping its length
reasonably brief?

**Keyword score (max. 6 points)**: each keyword in $[k_1, k_2, k_3]$ is given
+2 points if relevant, -2 points if not relevant. The final keyword score $K_i$
for chunk $c_i$ is the sum of the scores of all keywords, or:

$$
K_i = 2 * (N_{relevant} - N_{irrelevant} )
$$

**Latency score (max. 10 points)**: chunk $c_i$ will be assigned a latency
bonus $L_i$ that depends on how big its processing time was. The latency bonus
only applies for chunks that have a base score of at least 10. Whilst the value
of this bonus score is capped at 10, it is reasonable to expect that a very
fast pipeline would produce a chunk summary after at least 1-2s, thus setting
the maximum value of the latency score at around 6.

$$
L_i = \begin{cases}
0 & \text{if } B_i < 10 \\
10 e^{-0.5 * proc_i} & \text{if } B_i \ge 10
\end{cases}
$$

**Janus bonus (4 points)**: submissions that adopt Janus instead of the proposed
`ffmpeg` pipeline to produce the source audio for the pipeline will be assigned
a flat score of 4 points that will be added to the overall score.

---

The final chunk score $C_i$ is then:

$$
C_{i} = B_i + K_i + L_i
$$

The final score for an audio source is the average of all scores of its chunks.
Submission scores will then be processed through min-max normalisation.

## Evaluation environment

All the submitted pipelines that require GPU capabilities will be tested using a
single **RTX Pro 4500** card. The card will be used to run all the models
adopted in the pipeline, so make sure your solution fits in there!

## References and Resources

- Juturna framework: https://github.com/meetecho/juturna
- Janus WebRTC server: https://github.com/meetecho/janus-gateway
