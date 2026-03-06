# The STREAMIND challenge

Welcome to the STREAMING Grand Challenge!

The objective of the challenge is to design and implement a complete streaming
pipeline, bridging the gap between live audio reception and structured language
model inference. Participants will build their solutions on top of
[Juturna](https://github.com/meetecho/juturna), an open-source Python framework
developed by [Meetecho](https://www.meetecho.com/en/) for real-time AI data
pipeline prototyping.

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

## Evaluation metrics

All submitted chunks for an audio source will be assigned a composite score.

Chunk $i$, associated with the reference summary $ref_i$, contains the generated
summary $gen_i$ and the keyword list $[k_1, k_2, k_3]_i$. Its summary score
$S_i$ can then be computed using the LLM-as-a-judge technique, and normalized
between 0 and 30.

$$
S_i = \text{Judge}(ref_i, gen_i, [k_1, k_2, k_3]_i) : 0 <= S_i <= 30
$$

The final chunk score $C_i$ is obtained by subtracting the latency associated
with the chunk $i$ from its summary score.

$$
C_{i} = S_i - {\text{latency}_i}
$$

The final score for an audio source is the average of all scores of its chunks.

## References and Resources

- Juturna framework: https://github.com/meetecho/juturna
- Janus WebRTC server: https://github.com/meetecho/janus-gateway
