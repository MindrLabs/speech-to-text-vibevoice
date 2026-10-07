[English](README.md) · [한국어](README.ko.md) · [中文](README.zh-cn.md)

# speech-to-text-vibevoice

A speech recognition service: send a recording, get a transcript back. With enough memory, a meeting close to an hour long goes in whole, without cutting it into pieces.

We served Microsoft's VibeVoice-ASR (8.7B) with model-compose and ran it on an RTX 4090 and an NVIDIA DGX Spark. We asked whether it runs on our hardware and, if so, what it does well and badly. We compared it with another model, Whisper large-v3, only on short read sentences (Section 3.4). The original weights do not fit on an Apple M2 16GB, so we ran the same recordings there with a third-party 4-bit MLX conversion and compared the results with the original (Section 2). We also tested the streaming model, VibeVoice-ASR-Streaming (Section 3.6).

There were three kinds of input: Apollo 11 air-to-ground transcripts (NASA, public domain) read aloud by a TTS model, a public meeting recording (AMI), and NASA's public Artemis II press conference and 1962 radio recordings. To let English, Korean and Chinese readers each find results for their own language, we also ran the same read sentences (FLEURS) in all three languages, and gave hotwords, code-switching and numbers the same tests in each language.

A language model drafted the verdicts by reading each output next to the reference script, one author confirmed them, and we also measured character error rate (CER) and word error rate (WER).

![A 26-second recording uploaded to the web UI; on Run, English radio calls and their Korean interpretation come back as segments with speaker IDs and times](docs/images/shared/demo.gif)

Demo: a 26-second recording (TTS) that alternates English radio calls and Korean interpretation, transcribed in the web UI (gradio). It took 5.4 s on an RTX 4090, shown at real speed. The server ran with the same settings as this repo's defaults (web UI, segment output).

## Contents

- [Quick start](#quick-start)
- [Summary](#summary)
- [1 Setup](#1-setup)
- [2 Does it run on my hardware?](#2-does-it-run-on-my-hardware)
  - [A 16 GB Mac runs it in 4-bit](#a-16-gb-mac-runs-it-in-4-bit)
- [3 Results by feature](#3-results-by-feature)
  - [3.1 60-minute single pass](#31-60-minute-single-pass)
  - [3.2 Speaker diarization and timestamps](#32-speaker-diarization-and-timestamps)
  - [3.3 Hotwords](#33-hotwords)
  - [3.4 Accuracy by language (no language setting)](#34-accuracy-by-language-no-language-setting)
  - [3.5 Code-switching](#35-code-switching)
  - [3.6 The streaming model (VibeVoice-ASR-Streaming)](#36-the-streaming-model-vibevoice-asr-streaming)
  - [3.7 Numbers](#37-numbers)
- [4 Recommendations](#4-recommendations)
- [5 Limits and what we did not measure](#5-limits-and-what-we-did-not-measure)
- [License](#license)

## Model

[`microsoft/VibeVoice-ASR`](https://huggingface.co/microsoft/VibeVoice-ASR) is a speech recognition model that transcribes a recording and also says who spoke (speaker) and when (timestamps). The weights are 17.3 GB.
It compresses speech to 7.5 tokens per second and feeds it to an LLM (based on Qwen2.5), so it can take up to 60 minutes of audio in one pass. It supports over 50 languages and takes no language setting.

## Quick start

You need [model-compose](https://github.com/hanyeol/model-compose): `pip install model-compose`.

Start it from one config file (`model-compose.yml`):

```bash
model-compose up
```

Once the model loads, a web UI opens at `http://localhost:8102`. Upload a recording and you get a segment list with speaker numbers and times. The HTTP API is at `http://localhost:8101/api`. To change ports, copy `.env.sample` to `.env` and edit `PORT` and `SERVER_PORT`. The first start installs torch, which takes from a few minutes to tens of minutes (that is why the config sets `start_timeout: 30m`).

Example request:

```bash
curl -o out.json localhost:8101/api/workflows/runs \
  -F workflow_id=transcribe -F wait_for_completion=true -F output_only=true \
  -F "input.audio=@meeting.wav" \
  -F "input.context_info=Tranquility Base, Eagle"
```

`context_info` is a list of names and terms (hotwords) and may be empty. The result is a segment list (`{text, start_time, end_time, speaker_id}`). The report measured with transcript-only output (`return_timestamps: false`, workflow output `${output as text}`); only the speaker and time results in Section 3.2 were measured with segment output on.

**Running the streaming model.** Change four things in `model-compose.yml` and the result flows out chunk by chunk (Section 3.6).

- Set the component's `model` to `microsoft/VibeVoice-ASR-Streaming-7B`
- Set the workflow output to `${output as stream/text}`
- Set the action's `return_timestamps` to `false`
- Set the action's `streaming` to `true`

The request is the same as above; add `-N` to curl to print chunks as they arrive.

**On a 16 GB Mac.** The original weights (17.3 GB) do not fit on a 16 GB Mac. Run the third-party 4-bit conversion [`mlx-community/VibeVoice-ASR-4bit`](https://huggingface.co/mlx-community/VibeVoice-ASR-4bit) (5.7 GB) directly with [mlx-audio](https://github.com/Blaizzy/mlx-audio); model-compose has no MLX driver. mlx-audio pins a pre-release transformers, so it gets its own virtual environment (needs [uv](https://docs.astral.sh/uv/)).

```bash
uv venv .venv-mlx --python 3.12
uv pip install --python .venv-mlx/bin/python --prerelease=allow "mlx-audio==0.3.0"
.venv-mlx/bin/mlx_audio.stt.generate --model mlx-community/VibeVoice-ASR-4bit \
  --audio meeting.wav --output-path out --format json
```

`out.json` holds the segment list with speaker numbers and times. Pass hotwords with `--context "Tranquility Base, Eagle"`.

## Summary

Every result below is a single run except the 4090 processing times (three runs), and most inputs are synthetic (TTS) or read speech. Treat CER differences of 0.01–0.02 as noise.

1. A 24 GB RTX 4090 handled recordings up to 35 minutes; 39 minutes ran out of memory. Sending long recordings back to back to the same server ran out of memory even at 30 minutes ([Section 2](#2-does-it-run-on-my-hardware)).
2. On the same 10-minute recording, the 4090 was about 4.6× faster than the DGX Spark. A 16 GB Mac runs a 4-bit conversion that wrote nearly the same text, more slowly ([Section 2](#2-does-it-run-on-my-hardware)).
3. The DGX Spark transcribed a 59-minute recording in one pass and got the speaker count of a 4-person meeting right ([3.1](#31-60-minute-single-pass), [3.2](#32-speaker-diarization-and-timestamps)).
4. On the same read sentences, English was most accurate, then Chinese, then Korean. Korean and Chinese sometimes lost whole sentences. Whisper large-v3, given the language, lost none of the same sentences and was more accurate on Korean and on noisy audio ([3.4](#34-accuracy-by-language-no-language-setting)).
5. Hotwords fixed Korean names, in Korean and in English sentences, but did not reliably fix English numbers or acronyms ([3.3](#33-hotwords)).
6. English terms inside Korean sentences came out transliterated into Hangul even with hotwords; inside Chinese sentences they stayed in English ([3.5](#35-code-switching)).
7. Number forms were not consistent: words, digits, and in English radio calls even Chinese characters ([3.7](#37-numbers)).
8. On noisy old radio it repeated one phrase until cut off, and the server then returned an empty result with no error ([Section 5](#5-limits-and-what-we-did-not-measure)).
9. The streaming model showed first text in 0.7–2.3 s and separated the 4 speakers of a real meeting, but merged four TTS voices into one ([3.6](#36-the-streaming-model-vibevoice-asr-streaming)).

Table 1: Summary by device

| Device | Runs? | Same 10-min recording | Peak memory |
|---|---|---|---|
| Apple M2 16GB | The original does not fit. Runs with a 4-bit conversion | 7 min 9 s (RTF 0.71, 4-bit) | 11.4 GB (with swap) |
| RTX 4090 24GB | Up to 35 min. 39 min ran out of memory | 1 min 20 s (RTF 0.13) | 21.3 GB |
| DGX Spark | Up to 59 min | 6 min 7 s (RTF 0.61) | 21.3 GB (28.1 GB at 59 min) |

The same 10-minute recording is an excerpt of the Artemis II press conference, run once on each device (methods and memory measures in Table 3). The Mac ran without hotwords. The 4090 and DGX runs used hotwords, but on the DGX hotwords barely changed the time (366.6 s vs. 369.3 s).

Table 2: Summary by language

| | English | Korean | Chinese |
|---|---|---|---|
| Read sentences, clean ([3.4](#34-accuracy-by-language-no-language-setting)) | Missed 0 of 19, CER 0.009 | Missed 3 of 20, CER 0.032 | Missed 2 of 19, CER 0.025 |
| Read sentences, 5 dB noise | Missed 1 of 19, CER 0.141 | Missed 6 of 20, CER 0.211 | Missed 3 of 19, CER 0.141 |
| Same sentences, Whisper large-v3 (language set), clean / noise | Missed 0, CER 0.010 / 0.076 | Missed 0, CER 0.013 / 0.070 | Missed 0, CER 0.035 / 0.119 |
| Names and hotwords ([3.3](#33-hotwords)) | English names right without hotwords. Korean names in English sentences wrong, fixed by hotwords | Names wrong, fixed by hotwords (CER 0.021 → 0.000) | Names right without hotwords. Only the first word of a recording was wrong, possibly unclear audio |
| English terms in sentences ([3.5](#35-code-switching)) | - | Transliterated into Hangul, even with hotwords | Stayed in English |
| Numbers ([3.7](#37-numbers)) | Mostly words, dates and times in digits. Some Chinese characters in radio calls | Hangul numerals. Alarm codes came out wrong | Chinese numerals, values right |
| Long recordings and speakers ([3.1](#31-60-minute-single-pass), [3.2](#32-speaker-diarization-and-timestamps)) | 59 min in one pass. 4 speakers of a meeting right | Tested up to 1 min 21 s only | Not tested |

CER is the mean over transcribed sentences, with non-speech tags such as `[Silence]` removed. Readings are FLEURS sentences, the same content in all three languages, run once on the DGX.

## 1 Setup

Table 3: Devices and methods. Look here for how each result was measured.

| Device | How it ran | Memory measure | Results |
|---|---|---|---|
| RTX 4090 (24 GB, shared) | model-compose 0.4.109 server, when nothing else was running | Whole-GPU usage | Table 4 and the maximum-length table in Section 2, Tables 10 (except DGX rows), 11, 13 (Korean rows), 14, the Whisper rows of Table 12, Figures 5, 6, 8 |
| RTX 4090 | The same direct-call script as the DGX | PyTorch peak | The 4090 row of Table 1, Figure 1 |
| DGX Spark (GB10, 120 GB unified, shared) | Script that calls transformers directly | PyTorch peak | Tables 1, 5, 9, 16, the original-model values in Table 6, Figures 1–4, 7 |
| DGX Spark | model-compose 0.4.109 server, when nothing else was running | Not measured | Three-language tests: DGX rows of Tables 10 and 13, VibeVoice rows of Table 12, Table 15 |
| MacBook (Apple M2, 16 GB unified) | mlx-audio 0.3.0 script (4-bit conversion), with other apps open and 8–10 GB of swap in use | MLX peak | The Mac row of Table 1, Table 6 |
| MacBook (Apple M2) | The same script wrapped in a model-compose server shell component | Not measured | Table 7, silence/noise tests |

Do not compare values with different memory measures. Whole-GPU usage comes out higher than PyTorch's peak even for the same run.

- Weights: `microsoft/VibeVoice-ASR` revision `d0c9efdb`, bf16, sdpa attention, greedy decoding (temperature 0, beam 1).
- The Mac alone ran the third-party 4-bit conversion `mlx-community/VibeVoice-ASR-4bit` (revision `a1a15cb6`) with mlx-audio 0.3.0 (mlx 0.32.3). Its times come from a script that loads the model once, so loading (about 5 s) is excluded.
- On the 4090, we started a server with model-compose 0.4.109 and sent requests. The first request after startup (warmup, on a short recording) and model loading (about 20 s) are not counted. Recordings up to 19 minutes ran three times each. From 25 minutes on, each length ran once on a freshly started server; these inputs are English TTS calls joined and cut to length. Memory grows with recording length, not with content.
- The comparison model, Whisper large-v3 (revision `06f233fe`), ran once per FLEURS clip on the 4090 through a model-compose 0.4.109 server (Hugging Face driver, default settings), with the language set for each clip.
- Inputs
  - 15 Apollo 11 recordings: NASA's 1969 transcripts (technical air-to-ground and public-affairs commentary) read by preset voices of Qwen3-TTS CustomVoice. No real astronaut's voice was used or imitated. The Korean and Japanese scripts are edited translations of the originals. We did not use VibeVoice-family TTS, since part of this model's training data is VibeVoice TTS output and that could flatter the results.
  - For the 4090 length limit: four recordings made by joining these English calls with 0.5 s pauses and cutting them at 25, 30, 35 and 39 minutes. Some passages repeat.
  - AMI meeting ES2004c (4 speakers, 38 min 54 s): public data that recorded the same meeting with headsets (IHM) and a single table microphone (SDM) at once.
  - The Artemis II post-flight press conference (public NASA video, 59 min 24 s) and its first 10 minutes as an excerpt.
  - 1962 Friendship 7 radio clips (public NASA audio, 45 s and 100 s) and noise recordings (silence, white, pink, radio band; 90 s each).
  - Mac only: 20 Korean read sentences from FLEURS (6–14 s each), as recorded and mixed with white noise at 5 dB SNR, twice each. Also 30 s of silence, 60 s of synthetic noise, and a read sentence followed by silence.
  - DGX, three languages: the same 20 Korean FLEURS sentences on the original model, plus the Chinese (Mandarin) and English readings of the same sentences. FLEURS reads one set of sentences in every language, so all three languages read the same content. One sentence (no. 1888) has digits or Latin letters in its Chinese and English scripts, so those two have 19. Clean and with white noise at 5 dB SNR, once each.
  - DGX, eight more TTS recordings: Chinese versions of the Korean hotword and code-switching scripts (voices serena and uncle_fu), English sentences with Korean and Chinese names, and the same seven number sentences in Korean, English and Chinese. A language model wrote these scripts. We transcribed every sentence with Whisper to check that the TTS read it as written. Of the sentences Whisper got wrong, the author listened to the Korean ones; the Chinese ones were checked again with a Chinese-specific recognizer (FunASR Paraformer), and no native speaker listened to them.
- Verdicts: a language model (Claude) read each output next to the reference script twice, independently, and drafted a verdict. It read text only and could not hear the audio. The author confirmed the drafts while listening; 93% of drafts (14 of 15) were kept as written.
- Scores: CER and WER are computed after removing punctuation, spaces and case. Tags the model adds for non-speech, such as `[Silence]`, are removed before scoring; a sentence that came back as tags only is counted as missed. Chinese has no spaces between words, so we report CER only. Number forms ("7" vs "seven") are not normalized, so English radio calls score worse than they really are. For English we therefore also report WER with both sides normalized by the Whisper English normalizer.

## 2 Does it run on my hardware?

Table 4: RTX 4090 processing time by recording length (median of 3 runs)

| Recording | Length | Processing time | Spread of 3 runs | RTF | Peak VRAM |
|---|---|---|---|---|---|
| Korean status report | 24.5 s | 2.6 s | 0.1% | 0.11 | 17.4 GB |
| Korean status report | 1 min 21 s | 9.9 s | 0.4% | 0.12 | 21.4 GB |
| English launch calls | 2 min 23 s | 22.1 s | 0.5% | 0.15 | 21.5 GB |
| English powered-descent calls | 5 min 8 s | 38.7 s | 2.7% | 0.13 | 21.5 GB |
| English landing calls | 9 min 6 s | 1 min 40 s | 1.1% | 0.18 | 22.7 GB |
| English first-EVA calls | 18 min 58 s | 4 min 13 s | 0.4% | 0.22 | 22.9 GB |

**RTF.** Processing time divided by recording length. Below 1 means it finishes faster than listening to the recording. The model writes the transcript one token at a time, and the longer the recording, the more input each new token has to attend to, so RTF grows. The 9-minute recording took 100 s and the 19-minute one 253 s: about twice the length, 2.5 times the time. 35 minutes took 660 s (RTF 0.31).

- Hotwords barely changed processing time (English descent calls, one run each: 39.7 s → 39.1 s).
- Three runs of the same recording were within 3% of each other.
- Spread of 3 runs is (longest − shortest) ÷ median.

**Maximum length.** Recordings made by joining English calls and cutting them to length, each length sent once to a freshly started server.

| Length | Result | Processing time | RTF | Peak VRAM |
|---|---|---|---|---|
| 25 min | Finished | 6 min 34 s | 0.26 | 22.1 GB |
| 30 min | Finished | 8 min 49 s | 0.29 | 23.0 GB |
| 35 min | Finished | 11 min 0 s | 0.31 | 23.0 GB |
| 39 min | Out of memory | - | - | - |

These ran on fresh servers, so their VRAM (whole-GPU usage) can come out lower than the 19-minute row of Table 4, which was measured after several requests on the same server.

- The 4090 (24 GB) fit recordings up to 35 minutes (23.0 GB). At 39 minutes it ran out of memory trying to allocate another 5.0 GB. The limit lies between 35 and 39 minutes.
- Long recordings sent back to back to the same server fail at shorter lengths. A 30-minute recording sent right after a 25-minute one on the same server ran out of memory, because PyTorch was still holding 3.5 GB it had freed but not released. On a fresh server the same 30 minutes ran to the end.

Table 5: DGX Spark processing time by recording length (direct call, one run each)

| Recording | Length | Processing time | RTF | Peak memory |
|---|---|---|---|---|
| Artemis II press conference, excerpt | 10 min | 6 min 7 s | 0.61 | 21.3 GB |
| AMI meeting (table mic) | 38 min 54 s | 50 min 36 s | 1.30 | 24.4 GB |
| AMI meeting (headset) | 38 min 54 s | 51 min 30 s | 1.32 | 24.4 GB |
| Artemis II press conference, full | 59 min 24 s | 85 min 36 s | 1.44 | 28.1 GB |

- On the DGX, the 10-minute recording finished faster than real time, and the 39- and 59-minute recordings took longer than the recordings themselves. We did not measure the lengths in between, so we do not know where it crosses over.

**Same-file comparison.** Tables 4 and 5 use different recordings and different measurement methods, so they cannot be compared across devices. We therefore ran two of the DGX recordings on the 4090 as well, once each, with the same script and settings.

![Processing time for the same two recordings: the 10-minute excerpt took 80 s on the RTX 4090 and 6.1 min on the DGX Spark. The 39-minute AMI meeting ran out of memory on the RTX 4090 and took 51.5 min on the DGX Spark](docs/images/en/chart-same-file.png)

Figure 1: Time to transcribe the same recordings on both devices.

- The 10-minute excerpt took 80 s on the 4090 and 367 s on the DGX, about 4.6× faster. Generated tokens per second were 41.5 vs 8.9.
- The 39-minute meeting ran out of memory on the 4090 14 s after it started. That run came right after the 10-minute excerpt in the same process, but a 39-minute TTS recording alone on a fresh server also ran out of memory. On the DGX it used 24.4 GB and took 51 min 30 s.
- Peak memory for the 10-minute excerpt was 21.3 GB on both. The transcripts had nearly the same number of words (1,986 vs 1,983).
- On the same 4090, this 10-minute recording (RTF 0.13) was faster than the 9-minute TTS calls in Table 4 (0.18). Both the content and the method (script vs model-compose server) differ, so we could not separate the cause; do not mix the values in Table 4 and Figure 1.

### A 16 GB Mac runs it in 4-bit

The 4-bit MLX conversion of the weights is 5.7 GB (the original is 17.3 GB). With it, we transcribed three recordings that the original had also processed and set the results side by side.

Table 6: Apple M2 16GB in 4-bit vs. the original (one run each)

| Recording | Length | Mac time | Mac peak memory | Speakers (truth / original / Mac) | WER original | WER Mac |
|---|---|---|---|---|---|---|
| AMI meeting (headset), 19:00–20:00 | 1 min | 1 min 28 s (RTF 1.47) | 7.7 GB | 4 / 4 / 2 | 0.301 | 0.277 |
| AMI meeting (headset), 19:00–24:00 | 5 min | 5 min 18 s (RTF 1.06) | 9.6 GB | 4 / 4 / 4 | 0.229 | 0.230 |
| Artemis II press conference excerpt | 10 min | 7 min 9 s (RTF 0.71) | 11.4 GB | no reference / 9 / 10 | - | 2% different from the original |

All original values are DGX results. The two AMI rows cut the same window out of a single pass over the whole meeting (39 min), and WER is measured as in Section 3.6. The 10-minute excerpt has no reference transcript, so we measured how much the Mac's words differ from the original's DGX output under the same condition (no hotwords). Mac times exclude model loading.

- **The transcripts were nearly the same as the original's.** The 10-minute excerpt was 1,995 vs. 2,009 words, and 2% of the words differed (Table 6). AMI 5 min had WER 0.230 vs. 0.229. For the English meeting and press conference, going to 4-bit barely changed the results.
- **Speaker separation wobbled on a short clip.** On AMI 1 min it merged 4 speakers into 2. On the 5-minute window with the same 4 people, it separated all 4, like the original. On the 10-minute excerpt the original found 9 speakers and the Mac 10.
- **Short recordings were slower than real time.** The 1-minute clip took 1.5× its length. RTF fell as recordings got longer: 5 minutes took about as long as the recording, and 10 minutes finished in 7 min 9 s, faster than the recording. Without hotwords, the DGX took 6 min 9 s on the same 10 minutes (the 4090 only has a run with hotwords: 1 min 20 s).
- **Memory grew from 7.7 GB to 11.4 GB with recording length.** 8–10 GB of swap was in use during the runs. Closing other apps may make it faster.

**Short Korean sentences.** We sent 20 Korean read sentences from FLEURS to the model-compose server twice each. Both runs gave identical results.

Table 7: FLEURS Korean, 20 sentences (Mac, 4-bit)

| Condition | Transcribed | Missed entirely | CER of transcribed sentences (median / mean) | Exactly right |
|---|---|---|---|---|
| Clean | 15 | 5 | 0.000 / 0.029 | 8 |
| White noise, 5 dB SNR | 13 | 7 | 0.222 / 0.328 | 0 |

- **Transcribed sentences were mostly accurate.** 8 of the 15 clean sentences had no errors apart from spacing and punctuation. Most errors were in foreign proper nouns ("카사블랑카" Casablanca → "가사 블랑카", "듀발" Duval → "쥐발", "피히테" Fichte → "피히트의"). The worst sentence wrote "플리트비체 호수 국립공원은" (Plitvice Lakes National Park) as "플리프 빛의 호소 공익공원은" (CER 0.189).
- **A missed sentence came back as a single `[Silence]` or `[Music]` tag.** These were read sentences of 8–14 s, and loudness did not explain it: one missed sentence was louder than most of the transcribed ones. Two missed sentences gave the same result when fed to the script directly, without model-compose. The original (DGX) also missed 3 of the same 20 clean sentences, but only one of them (no. 1879) was the same sentence, so missing whole sentences is not caused by 4-bit alone (Section 3.4).
- **Noise made it much worse.** Missed sentences rose to 7, and the 13 transcribed ones had a median CER of 0.222. "낭만주의는" (Romanticism) became "남만주 의", and the Plitvice sentence turned into an unrelated one.
- **It did not invent speech from silence or noise.** 30 s of silence and 60 s of synthetic noise each produced a single `[Silence]`. A read sentence followed by silence was transcribed and then followed only by `[Silence]`. This matches the original's noise test (Section 5).

## 3 Results by feature

Table 8: Features the official materials highlight

| Official feature | Our result |
|---|---|
| 60-minute single pass | Transcribed 35-minute (4090) and 59-minute (DGX) recordings end to end without cutting them. The 4090 (24 GB) ran out of memory on a 39-minute recording. |
| Speaker diarization and timestamps | Got the speaker count right in a 4-person meeting, with DER 11.3% (headset). A press conference with changing questioners was split into 25 speakers. |
| Custom hotwords | Korean names and places were fixed, in Korean and in English sentences. Chinese names were right without them. English numbers and acronyms barely changed. |
| 50+ languages, no language setting | Wrote Korean, English, Chinese and Japanese in their own scripts. On the same read sentences, English was most accurate, then Chinese, then Korean, and none beat Whisper large-v3 given the language. Japanese had more errors, with CER 0.169. |
| Code-switching | Followed recordings that switch language line by line. English terms inside Korean sentences were transliterated into Hangul; inside Chinese sentences they stayed in English. |
| Streaming (separate checkpoint) | First text came within 0.7–2.3 s, and it separated the 4 speakers of a real meeting. It merged four TTS voices into one speaker. |

### 3.1 60-minute single pass

> Official GitHub: "VibeVoice ASR accepts up to 60 minutes of continuous audio input within 64K token length."

- **4090, 18 min 58 s EVA calls:** went in whole and was transcribed to the end. The output starts with "Okay, Houston, I'm on the porch.", passes through "That's one small step for man, one giant leap for mankind." and ends with "We've got this view, Neil." No phrase was repeated.
- **DGX, 59 min 24 s Artemis II post-flight press conference:** 230 segments and about 10,800 words, with the last segment running to the end of the recording (59:23). Music and room noise at the start and end were tagged separately as `[Music]` and `[Environmental Sounds]`. Peak memory was 28.1 GB.
- **4090, 38 min 54 s AMI meeting:** ran out of memory right after it started. On the 4090, joined TTS calls ran to the end up to 35 minutes and ran out of memory at 39 (Section 2). Putting a recording close to 60 minutes through in one pass needs more than 24 GB of memory.

### 3.2 Speaker diarization and timestamps

The 4090 runs were set to return only the transcript text, so they did not measure speakers or times. The results below come from the DGX with segment output on.

Table 9: AMI meeting ES2004c (4 speakers, 38 min 54 s), recorded with two kinds of microphone at once.

| Metric (%, lower is better) | Headset | Table mic | Paper, headset | Paper, table mic |
|---|---|---|---|---|
| WER (word error rate) | 17.12 | 21.37 | 18.81 | 24.65 |
| cpWER (WER grouped by speaker) | 15.50 | 20.45 | 20.41 | 28.82 |
| tcpWER (WER that also requires the right time, ±5 s) | 15.74 | 21.09 | 20.82 | 29.80 |
| DER (diarization error, ±0.25 s collar) | 11.31 | 17.51 | 11.92 | 13.43 |
| Speaker count (reference 4) | 4 | 4 | - | - |

![Error rates for headset vs a single table mic: WER 17.1 vs 21.4, cpWER 15.5 vs 20.4, tcpWER 15.7 vs 21.1, DER 11.3 vs 17.5](docs/images/en/chart-mic.png)

Figure 2 (DGX): Error rates for the same meeting recorded with different microphones.

- In a meeting where the same 4 people talked for all 39 minutes, both microphones got the speaker count right. tcpWER was only 0.2–0.6 pp above cpWER, so the times were right at least to within the metric's ±5 s window.

![Speaker timeline of a 4-person meeting: for each speaker, the top lane is the reference and the bottom lane is the model, and they are filled at nearly the same times](docs/images/en/speakers-ami.png)

Figure 3 (DGX): Speaker timeline of the 39-minute AMI meeting. For each speaker, the top lane is the reference and the bottom lane is the model's segments.

- With a single table microphone, WER rose by 4.3 pp and DER by 6.2 pp. That is close to recording a meeting with one laptop or phone.
- In the 59-minute press conference with changing questioners, it found 25 speakers. 18 of them have only 1–3 segments, so it likely split the same person into several (there is no reference).

![Timeline of the 59-minute press conference split into 25 speakers: the top 7 have many segments and the bottom 18 have only 1–3](docs/images/en/speakers-artemis.png)

Figure 4 (DGX): Artemis II post-flight press conference, 59 minutes. Speakers found by the model, ordered by segment count. Gray marks speakers with only 1–3 segments.

- The paper's numbers cover a whole test set and ours a single meeting, so read them only as the same ballpark. DER nearly doubles depending on the collar (20.25% with no collar). AMI is widely used public data, so we cannot rule out that it was in the pretraining data.

### 3.3 Hotwords

> Model card: "Users can provide customized hotwords (e.g., specific names, technical terms, or background info) to guide the recognition process, significantly improving accuracy on domain-specific content."

We ran each recording once without and once with hotwords.

![A Korean lunar-surface report transcribed without and with hotwords: without them, 닐 (Neil) became 네 ("yes") and 메사 (mesa) became 매사 ("everything")](docs/images/en/card-hotword.png)

Figure 5 (4090): Before and after hotwords. Differences in spacing and commas are not marked, matching how scores are computed.

Table 10: Score changes with and without hotwords (4090; rows marked DGX ran on the DGX)

| Recording | Hotwords | CER | WER |
|---|---|---|---|
| Korean lunar-surface report | 닐, 버즈, 메사, 비상 시료, 휴스턴 (Neil, Buzz, mesa, contingency sample, Houston) | 0.021 → **0.000** | 0.278 → 0.056 |
| Korean landing calls | 이글, 컬럼비아, 트랭퀼리티 베이스, 프로그램 알람, 동력 하강 (Eagle, Columbia, Tranquility Base, program alarm, powered descent) | 0.056 → **0.016** | 0.184 → 0.053 |
| English powered-descent calls | Eagle, Columbia, Tranquility Base, DELTA-H, PGNS, AGS, P64 | 0.387 → 0.390 | 0.376 → 0.357 |
| Chinese lunar-surface report (DGX) | 尼尔, 巴兹, 台地, 应急样本, 休斯敦 (Neil, Buzz, mesa, contingency sample, Houston) | 0.027 → **0.000** | - |
| Chinese landing calls (DGX) | 鹰号, 哥伦比亚号, 静海基地, 程序警报, 动力下降 (Eagle, Columbia, Tranquility Base, program alarm, powered descent) | 0.022 → 0.022 | - |
| English sentences with Korean and Chinese names (DGX) | Yi So-yeon, Nuri, Naro Space Center, Goheung, Danuri, Yang Liwei, Jiuquan, Wang Yaping, Zhai Zhigang, Tiangong, Chang'e, Tianwen, Wenchang | 0.058 → 0.024 | 0.131 → 0.048 |

Measured again with WER that also normalizes number forms, the English descent calls went from 22.6% to 20.3%, a small drop.

- **In the Korean lunar-surface report, hotwords got every name right.** Without them, "닐, 지금 메사 쪽" ("Neil, toward the mesa now") came out as "네, 지금 매사 쪽" ("Yes, now everything …"). With hotwords, "닐" (Neil) and "메사" (mesa) matched the script. "버즈" (Buzz) and "비상 시료" (contingency sample) were already right without hotwords (only the spacing differed, "비상시료").
- **The Korean landing calls were only partly fixed.** "슈스턴" became "휴스턴" (Houston) and "콜럼비아" became "컬럼비아" (Columbia), but "트랭퀼리티 베이스" (Tranquility Base), which was on the list, came out as "트랭큘리티 베이스".
- **The English descent calls barely changed.** "Eagle" (9 times) and "Tranquility Base" already matched the script without hotwords, so there was nothing to fix. The errors were in numbers and acronyms. "Both odd" (both AUTO) and "Fuel stand is in" (413 is in) were the same with or without hotwords. Among the listed acronyms, "AGS" became correct, and "P64" was right in only one of two places (the other was "P六十" without hotwords and "P60" with them).
- **In Chinese, only the first word of each recording was wrong, and that may be the audio.** Without hotwords, the opening "尼尔" (Neil) came out as "你啊" ("you, ah") and the opening "鹰号" (Eagle) as "你好" ("hello"); with hotwords, "尼尔" was fixed but "鹰号" became "调好" ("tuned"). The same names later in the recordings were right. Whisper and a Chinese-specific recognizer (Paraformer) also misheard the opening "鹰号" (as "也好" and "您好"), so the first syllable of the TTS audio may be unclear there, and we do not count that "鹰号" error as a model error. "尼尔" was not like this: hotwords fixed it. Every other Chinese name and term, such as "静海基地" (Tranquility Base), was right without hotwords.
- **In English sentences, Korean names were wrong and Chinese names mostly right.** Without hotwords, "Yi So-yeon" came out as "Yusou Yan", "Goheung" as "Gohang" and "Danuri" as "The Nuri"; with hotwords all three matched. Of the Chinese names, "Yang Liwei", "Zhai Zhigang", "Jiuquan" and "Wenchang" were right without hotwords and "Wang Yaping" ("Wang Yebing") was fixed by them. "Chang'e" came out as "Qinghai" with or without hotwords.
- Hotwords did not increase processing time.

### 3.4 Accuracy by language (no language setting)

> Model card: "It supports over 50 languages, requires no explicit language setting, and natively handles code-switching within and across utterances."

Table 11: Scores by language (4090, no language setting)

| Recording | CER | WER |
|---|---|---|
| Korean, one-speaker report | 0.025 | 0.240 |
| Korean, two-speaker exchange | 0.000 | 0.000 |
| English status report | 0.004 | 0.045 |
| Japanese, one-speaker report | 0.169 | 0.909* |

\* The reference has no spaces between words, so do not read this as a word-level metric.

- **Nothing was translated or written in the wrong script.** Korean came out in Hangul, English in Latin letters, Chinese in simplified characters (no traditional characters in any Chinese output), and Japanese in hiragana, katakana and kanji.
- **Japanese was recognized as Japanese, but with many word errors.** "アポロ管制センター" (Apollo control center) came out as "アポロ厳正センター", and "乗組員" (crew) as "ノグミン".
- The Korean script's "이십일 시간 삼십팔 분" (twenty-one hours thirty-eight minutes, spelled out) came out in digits, "21시간 38분". The same countdown came out as "10, 9, 8" in one run and "Ten, nine, eight" in another, so number forms are not consistent.

**The same sentences in three languages.** FLEURS has people read one set of sentences in every language, so we took the 20 Korean sentences from Section 2 and the Chinese and English readings of the same sentences, and ran them on the original model (DGX). For comparison, the same clips also went through Whisper large-v3 with the language set (4090). Each language has different readers, so only the content is the same.

Table 12: FLEURS read sentences in three languages (one run each; VibeVoice original model on the DGX, Whisper large-v3 with the language set on the 4090)

| Language | Audio | Model | Missed entirely | CER of transcribed sentences (median / mean) | Exactly right | WER (mean) |
|---|---|---|---|---|---|---|
| English (19) | Clean | VibeVoice | 0 | 0.000 / 0.009 | 14 | 0.023 |
| English (19) | Clean | Whisper | 0 | 0.000 / 0.010 | 13 | 0.032 |
| Chinese (19) | Clean | VibeVoice | 2 | 0.000 / 0.025 | 10 | - |
| Chinese (19) | Clean | Whisper | 0 | 0.000 / 0.035 | 10 | - |
| Korean (20) | Clean | VibeVoice | 3 | 0.021 / 0.032 | 7 | 0.144 |
| Korean (20) | Clean | Whisper | 0 | 0.000 / 0.013 | 16 | 0.097 |
| English (19) | White noise, 5 dB SNR | VibeVoice | 1 | 0.064 / 0.141 | 6 | 0.226 |
| English (19) | White noise, 5 dB SNR | Whisper | 0 | 0.010 / 0.076 | 9 | 0.115 |
| Chinese (19) | White noise, 5 dB SNR | VibeVoice | 3 | 0.112 / 0.141 | 3 | - |
| Chinese (19) | White noise, 5 dB SNR | Whisper | 0 | 0.042 / 0.119 | 8 | - |
| Korean (20) | White noise, 5 dB SNR | VibeVoice | 6 | 0.176 / 0.211 | 1 | 0.380 |
| Korean (20) | White noise, 5 dB SNR | Whisper | 0 | 0.048 / 0.070 | 7 | 0.231 |

Korean WER counts space-separated words (eojeol), so do not compare it with English WER.

- **On the same sentences, Whisper large-v3 did better.** Given the language, Whisper missed no sentence in any of the three languages. On Korean it led even on clean audio (mean CER 0.013 vs 0.032, 16 vs 7 sentences exactly right), and with noise it degraded less in all three languages. Clean English and Chinese differ by under 0.01, so we read them as the same. VibeVoice's CER leaves out the sentences it missed; counting them would widen the gap.
- **The comparison favors Whisper.** Whisper was told the language; VibeVoice has no such setting. We also tried Whisper without a language, but every request failed in model-compose 0.4.109 ("mel input features ... length 3000"), so it is not measured. The comparison covers only 6–14 s read sentences; features Whisper lacks, such as one-pass long recordings, speaker diarization and hotwords, were not compared. Whisper used 4.1 GB of GPU memory on these clips (VibeVoice: 17.4 GB on a 24.5 s recording, Table 4).
- **The order matches the training-data shares.** With clean audio, English was most accurate, then Chinese, then Korean. Of the training data, 66.7% is English, 14.4% Chinese and 0.9% Korean (paper appendix). With about 20 sentences per language read by different people, the gap between Chinese and Korean (CER 0.025 vs 0.032) may be noise; English's lead is clearer. With noise, every language got worse, Korean the most.
- **Whole sentences went missing in Chinese and Korean too.** A missed sentence came back as `[Silence]` only. Sentence no. 1755 was missed in both Korean and Chinese, but its English reading was transcribed. Loudness did not explain it: the Chinese reading of no. 1755 was louder than most transcribed sentences.
- **`[Silence]` tags were attached far more often in Korean and Chinese.** 33 of 40 Korean outputs and 25 of 38 Chinese outputs had a tag, usually `[Silence]` at the end, against 3 of 38 in English. The Korean and Chinese recordings have about 2 s of silence before and after the speech and the English ones about 0.7 s, so this may come from the recordings rather than the language. Remove these tags before showing or scoring the text: left in, a word-for-word correct Chinese transcript (no. 1660) scored CER 0.22.
- Besides these three and Japanese, we did not test any of the other 50+ languages.

### 3.5 Code-switching

Table 13: Code-switching recordings (Korean on the 4090, Chinese on the DGX)

| Recording | CER | Result |
|---|---|---|
| English original and Korean interpretation alternating line by line | 0.010 | Wrote English lines in English and Korean lines in Hangul, in order. Only "트랭퀼리티 베이스" (Tranquility Base) was wrong, as "트랜클리티 베이스" |
| English terms inside Korean sentences | 0.648 | Every English term was transliterated into Hangul, and some changed meaning |
| English original and Chinese interpretation alternating line by line | 0.000 | Wrote English lines in English and Chinese lines in Chinese, in order |
| English terms inside Chinese sentences | 0.000 | Every English term stayed in English. Only capitals were lost ("GO" → "go", "Eagle" → "eagle") |
| Same, with the terms as hotwords | 0.000 | Capitals matched the script too ("GO", "Eagle", "Tranquility Base") |

The Chinese recordings are the Korean scripts translated, with the same ten English terms in the same places.

English terms inside Korean sentences came out like this.

| Script | Output |
|---|---|
| powered descent | 파워드 디센트 (pawodeu disenteu) |
| landing radar가 lock-on 됐고 | 랜딩 레다라가 로군 됐고 (garbled) |
| program alarm | 프로그램 얼람 (peurogeuraem eollam) |
| Houston에서 바로 GO를 줬어요 | 휴스턴에서 바로 고를 줬어요 ("go" in Hangul) |
| Eagle이 Tranquility Base에 | 이 글이 트랜킬리티 베이스에 ("this writing" … Tranquility Base) |

![Script and output of a Korean landing briefing: every English term was transliterated into Hangul, with or without a term list](docs/images/en/card-code-switching.png)

Figure 6 (4090): Code-switching briefing. The bottom row is the follow-up run with the English terms given as hotwords.

**Follow-up: does a term list keep the terms in English?** We ran the same recording again with "Eagle, Tranquility Base, powered descent, landing radar, lock-on, program alarm, GO, manual attitude control, contact light, engine stop" as hotwords. Still not one term came out in Latin letters, and CER stayed at 0.648. Only the spellings shifted a little ("레다라가 로군" → "레이더라가 로건", "얼람" → "알람"). In sentences that are mainly Korean, a term list could not change which script the terms were written in.

**Chinese kept the English terms.** The same terms inside Chinese sentences came out in Latin letters, such as "现在开始进入 powered descent" and "landing radar 已经 lock on". The transliteration happened only in Korean sentences. For the opposite direction (other languages inside English sentences), we measured only Korean and Chinese names (Section 3.3).

### 3.6 The streaming model (VibeVoice-ASR-Streaming)

> Technical report: "produce 'who said what' as speech arrives, without a separate diarization stage."

The streaming model listens to the recording in chunks (2.9 s) and emits text as it goes. In model-compose, setting the workflow output to `${output as stream/text}` and the action to `streaming: true` makes the result flow out chunk by chunk. We ran the 7B (revision `60d858b5`) once each on four recordings (RTX 4090).

Table 14: Streaming 7B (4090, one run each)

| Recording | Actual speakers | Streaming speakers | First text | Done | WER streaming | WER non-streaming |
|---|---|---|---|---|---|---|
| English EVA calls, 2 min 47 s (TTS, 4 voices) | 4 | 1 | 1.8 s | 17.2 s | 0.035 | 0.023 |
| AMI meeting, 1 min (19:00–20:00) | 4 | 4 | 0.7 s | 9.3 s | 0.225 | 0.301 |
| AMI meeting, first 5 min | 4 | 4 | 2.3 s | 24.7 s | 0.182 | 0.159 |
| AMI meeting, 19:00–24:00 | 4 | 4 | 2.0 s | 45.1 s | 0.211 | 0.229 |

These WERs only normalize punctuation and case, so do not compare them directly with WERs in other tables. For non-streaming, AMI uses the same stretch of the 39-minute one-pass transcript (DGX), and EVA uses the 4090 measurement.

- **Results streamed right away.** The first text arrived in 0.7–2.3 s, and a 5-minute recording finished in 25–45 s (RTF 0.08–0.15). Given a file, it streams out as fast as it can rather than at the recording's pace. We did not test live input such as a microphone.
- **On real meetings it separated the speakers.** All three AMI stretches came out with 4 speakers. Speaker labels come inline in the text, like "Speaker 1:", and there are no timestamps.
- **Accuracy was close to the non-streaming model.** AMI WER was 0.18–0.23, close to the non-streaming result for the same stretches (0.16–0.30).
- **It merged the four TTS voices into one speaker.** The transcription was accurate (WER 0.035), but every line was Speaker 0. We did not check why it separated real people but not the TTS voices.
- We did not measure the 1.5B.

### 3.7 Numbers

The same seven number sentences (a date and time, a countdown, alarm codes 1202 and 1201, a decimal, an amount, a weight) were read in Korean, English and Chinese, one voice each, and run once on the DGX. The TTS read every number as words, so the table shows how the model chose to write what it heard.

Table 15: How numbers were written (DGX)

| Spoken | Korean output | English output | Chinese output |
|---|---|---|---|
| July 20, 1969, 4:17 p.m. | 천구백육십구년 칠월 이십일 오후 네시 십칠분 (words) | July 20th, 1969, at 4:17 p.m. (digits) | 一九六九年七月二十日下午四点十七分 (numerals) |
| Countdown 10 … 0 | 십구 팔 칠 … 영 ("10, 9" merged into "19") | Ten, nine, eight … zero | 十、九、八 … 零 |
| Alarm 1202, then 1201 | 일이공, 일이공에 (both wrong) | twelve o two, twelve o one | 一二零二, 一二零一 |
| 4.5 feet per second | 사점오피트 | four point five feet | 四点五英尺 |
| $25.4 billion | 이백오십사억 달러 | twenty five point four billion dollars | 二百五十四亿美元 |

- **The model wrote what it heard, mostly as words.** Only the English date and time came out in digits. Elsewhere it chose digits even for numbers read as words: a Korean "이십일 시간 삼십팔 분" came out as "21시간 38분" (Section 3.4). Do not expect one form.
- **The values were right except for Korean alarm codes.** "일이공이" (1-2-0-2) came out as "일이공" and "일이공일" (1-2-0-1) as "일이공에", and the Korean countdown merged "십, 구" (ten, nine) into "십구" (nineteen). English and Chinese got every value right.
- No Chinese characters appeared in these English sentences, unlike the English radio calls in Section 5.
- Even with the right values, these outputs score badly against a reference written in digits (CER 0.37–0.63), so normalize number forms before measuring accuracy.

## 4 Recommendations

- **Hardware:** A 24 GB GPU fits recordings up to 35 minutes; 39 minutes ran out of memory. A server that takes long recordings back to back runs out at shorter lengths (30 minutes failed right after 25). Run recordings over 39 minutes on a device with more memory (28.1 GB at 59 minutes). On the DGX Spark, from 39 minutes on, processing took longer than the recording. On a 10-minute recording, the 4090 was about 4.6× faster than the DGX Spark.
- **16 GB Mac:** The 4-bit conversion (mlx-audio) runs, and on a 10-minute English recording it produced nearly the same text as the original in a little over 7 minutes. It works for transcribing recordings ahead of time but is too slow for live captions. Short Korean recordings can come back entirely empty (`[Silence]`); the original model does this too, so when that happens, listen to the recording again before trusting it.
- **Meeting notes:** If you need speakers, get segment output with `return_timestamps: true`. In our one meeting, a single table microphone (close to recording with one laptop) raised WER by about 4 pp over headsets, and speaker attribution was shakier.
- **Hotwords:** Put names, places and in-house terms in hotwords. They did not add processing time and fixed Korean proper nouns, including Korean names inside English sentences. English numbers and acronyms (calls like P64 and AGS) were not reliably fixed, so have a person check them.
- **Code-switching:** In Chinese, English terms stay in English, and a term list also fixes their capitals. In Korean, if you need English terms written in English, keep a separate dictionary that maps the transliterations back ("파워드 디센트" → "powered descent"); this model did not write them in Latin letters even with a term list.
- **Non-speech tags:** Outputs often end with `[Silence]` or similar tags, especially on recordings with silence before and after the speech. Strip them before display or scoring, and treat an output that is tags only as "missed", not "nobody spoke".
- **Numbers:** The same number can come out as Arabic digits, words in the spoken language (Hangul or Chinese numerals, English words) or Chinese characters. Normalize them in post-processing when numbers matter, and normalize number forms before measuring accuracy.
- **Short recordings only:** If you do not need speakers, one-pass long recordings or hotwords, try Whisper large-v3 as well. On the same read sentences, Whisper given the language missed no sentence, was more accurate on Korean and on noisy audio, and used about a quarter of the GPU memory.
- **Streaming:** When you need results right away, use the streaming model. We fed it files only, so try it with live microphone input before using it for live captions. On the 4090 the first text came within 0.7–2.3 s and it separated the speakers of a real meeting. When you need the time of each line, give the non-streaming model the whole recording.
- **Old radio and noisy recordings:** A repetition loop can leave the result completely empty (Section 5). Do not read an empty result as "nobody spoke"; check the raw output, or set a token limit and a repetition check.

## 5 Limits and what we did not measure

### Repetition loops and silent empty results

Two 1962 Friendship 7 radio clips (45 s and 100 s) came back empty. The model did hear the speech: the start of the raw output was correct.

> Fifteen seconds. Godspeed, John Glenn. Ten, nine, eight, seven.

After that, in a noisy stretch, it repeated one phrase over and over ("Roger, turn around. Roger, turn around. …"). When the output is cut off at the token limit, the segment list (JSON) never closes, and the VibeVoice package's parser logs a warning and returns an empty list. A user can easily take that to mean "nobody spoke".

![Raw transcript of 45 s of 1962 radio: the start is correct, then it repeated "Roger, turn around" 280 times and was cut off, and the server result is an empty list](docs/images/en/card-loop.png)

Figure 7 (DGX): Repetition loop and empty result.

Table 16: Variables we changed (DGX)

| Variable | Values | Result |
|---|---|---|
| Hotwords | none; on (Friendship 7, Glenn, Cape Canaveral, …) | All 4 runs looped |
| Clip | 45 s, 100 s | Both correct at the start, looping later |
| Token limit | 1,500 | All 4 runs generated up to the limit, with no end token |
| Run path | model-compose server, direct call | Both empty |
| Input | noise only (silence, white, pink, radio band; 90 s each) | No loop. A single `[Silence]`, `[Noise]` or `[Environmental Sounds]` tag |
| Input | clean long recordings (AMI 39 min, press conference 59 min) | No loop, normal end |

With noise alone it did not make up sentences; the loops appeared when speech and noise were mixed. With a check that stops generation when the last 400 tokens repeat with a period of 100 tokens or less, these 4 runs were caught between 450 and 1,100 tokens, and the check never fired on normal output.

### Other limits

- **Most inputs are TTS recordings.** We saw real human speaking styles, overlapping speech and field noise only in the AMI meeting, the press conference and the 1962 radio. For overlapping speech, the paper itself says the model transcribes only the louder voice.
- **Chinese characters appeared in English radio calls.** "P64" became "P六十" and "1201" became "一八零一"; with hotwords, forms like "两千尺的" and "四十七 degrees" remained (Chinese characters only went from 20 to 15).

![Script and output of English descent calls: 1201 became 一八零一, 2000 feet became 两千feet的, 35 degrees became 三V五degrees, and P64 became P六十](docs/images/en/card-numbers.png)

Figure 8 (4090): Chinese numerals in English radio calls. 1201 came out as 一八零一 (1801), so even the value was wrong.

- **Korean transliterations of English place names were not stable.** "Tranquility Base" came out in four different Hangul spellings across four outputs, and hotwords did not pin it down (transliteration: Section 3.5).

- **The same input twice gives nearly the same output.** A 1-minute Korean recording gave identical results both times, and a 3-minute English recording differed in one punctuation mark out of 394 words ("inch. But" vs "inch, but").
- **Most conditions were measured once.** Only the 4090 processing times (up to 19 minutes) ran three times, and the three runs were within 3%. Each length in the maximum-length test ran once.
- **The Mac results come from a third-party 4-bit conversion.** We compared it with the original on three English recordings and the 20 Korean sentences (the original missed 3, the Mac 5). We measured with other apps open and swap in use, and did not test hotwords, code-switching or streaming on the Mac.
- **Chinese was tested on read sentences and TTS only.** We have no real Chinese conversation or meeting, so Chinese speaker diarization and long recordings are untested. The new three-language tests ran once each on the DGX, and their scripts were written by a language model.
- **Not measured:** comparisons with other models beyond short read sentences (Whisper and others on long recordings, meetings or hotwords), Whisper without a language set, the rest of the 50+ languages, Korean common words (not names) inside English sentences, the streaming 1.5B, live microphone input, the exact longest recording that fits on a 4090 (between 35 and 39 minutes), the speed of other inference engines such as vLLM, and a quantitative evaluation of overlapping speech.

## License

| Item | License | Commercial use |
|---|---|---|
| `microsoft/VibeVoice-ASR` weights | [MIT](https://huggingface.co/microsoft/VibeVoice-ASR/blob/main/LICENSE) | ✓ |
| `microsoft/VibeVoice-ASR-Streaming-7B` weights | [MIT](https://huggingface.co/microsoft/VibeVoice-ASR-Streaming-7B) | ✓ |
| `openai/whisper-large-v3` weights (comparison) | [Apache 2.0](https://huggingface.co/openai/whisper-large-v3) | ✓ |
| Apollo 11 transcripts (input scripts) | U.S. federal government work, public domain | ✓ |
| [AMI Meeting Corpus](https://groups.inf.ed.ac.uk/ami/corpus/) | CC BY 4.0 | ✓ (with attribution) |
| Public NASA video and audio (Artemis II press conference, Friendship 7 radio) | U.S. federal government work. Used only for measurement and not included in this repository | ✓ |
| [Qwen3-TTS CustomVoice](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice) (input speech synthesis) | Apache 2.0 | ✓ |
| `mlx-community/VibeVoice-ASR-4bit` conversion (for Mac) | [MIT](https://huggingface.co/mlx-community/VibeVoice-ASR-4bit) | ✓ |
| [FLEURS](https://huggingface.co/datasets/google/fleurs) Korean, Chinese and English (read-sentence input) | CC BY 4.0. Used for measurement only, not included in this repo | ✓ (with attribution) |

- This project is released under the MIT License.
- Microsoft does not recommend using VibeVoice in commercial or real-world applications without further testing and development.
- Per NASA's media usage guidelines, we do not use the NASA logo or imply NASA endorsement. Script sources: [Apollo 11 Technical Air-to-Ground Voice Transcription](https://archive.org/download/Apollo11Audio/AS11_TEC.pdf), [Apollo 11 Mission Commentary](https://archive.org/download/Apollo11Audio/AS11_PAO.PDF).
