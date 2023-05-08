# Stabilizing Timestamps for Whisper

This script modifies [OpenAI's Whisper](https://github.com/openai/whisper) to produce more reliable timestamps.

https://user-images.githubusercontent.com/28970749/225826345-ef7115db-51e4-4b23-aedd-069389b8ae43.mp4

* [Setup](#setup)
* [Usage](#usage)
  * [Transcribe](#transcribe)
  * [Output](#output)
  * [Regrouping Words](#regrouping-words)
  * [Locating Words]()
  * [Boosting Performance](#boosting-performance)
  * [Visualizing Suppression](#visualizing-suppression)
  * [Encode Comparison](#encode-comparison)
  * [Tips](#tips)
* [Quick 1.X → 2.X Guide](#quick-1x--2x-guide)

## Setup
```
pip install -U stable-ts
```

To install the latest commit:
```
pip install -U git+https://github.com/jianfch/stable-ts.git
```

## Usage
The following is a list of CLI usages each followed by their corresponding Python usages (if there is one). 

### Transcribe
```commandline
stable-ts audio.mp3 -o audio.srt
```
```python
import stable_whisper
model = stable_whisper.load_model('base')
result = model.transcribe('audio.mp3')
result.to_srt_vtt('audio.srt')
```
Parameters: 
[load_model()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/whisper_word_level.py#L637-L652), 
[transcribe()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/whisper_word_level.py#L75-L199)
### Output
Stable-ts supports various text output formats.
```python
result.to_srt_vtt('audio.srt') #SRT
result.to_srt_vtt('audio.vtt') #VTT
result.to_ass('audio.ass') #ASS
result.to_tsv('audio.tsv') #TSV
```
Parameters: 
[to_srt_vtt()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/text_output.py#L267-L291), 
[to_ass()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/text_output.py#L335-L353), 
[to_tsv()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/text_output.py#L401-L434)
<br /><br />
There are word-level and segment-level timestamps. All output formats support them. 
They also support will both levels simultaneously except TSV. 
By default, `segment_level` and `word_level` are both `True` for all the formats that support both simultaneously.<br /><br />
Examples in VTT.

Default: `segment_level=True` + `word_level=True` or `--segment_level true` + `--word_level true` for CLI
```
00:00:07.760 --> 00:00:09.900
But<00:00:07.860> when<00:00:08.040> you<00:00:08.280> arrived<00:00:08.580> at<00:00:08.800> that<00:00:09.000> distant<00:00:09.400> world,
```

`segment_level=True`  + `word_level=False` (Note: `segment_level=True` is default)
```
00:00:07.760 --> 00:00:09.900
But when you arrived at that distant world,
```

`segment_level=False` + `word_level=True` (Note: `word_level=True` is default)
```
00:00:07.760 --> 00:00:07.860
But

00:00:07.860 --> 00:00:08.040
when

00:00:08.040 --> 00:00:08.280
you

00:00:08.280 --> 00:00:08.580
arrived

...
```

#### JSON
The result can also be saved as a JSON file to preserve all the data for future reprocessing. 
This is useful for testing different sets of postprocessing arguments without the need to redo inference.
```commandline
stable-ts audio.mp3 -o audio.json
```
```python
# Save result as JSON:
result.save_as_json('audio.json')
```
Processing JSON file of the results into SRT.
```commandline
stable-ts audio.json -o audio.srt
```
```python
# Reload the result:
result = stable_whisper.WhisperResult('audio.json')
result.to_srt_vtt('audio.srt')
```

### Regrouping Words
Stable-ts has a preset for regrouping words into different segments with more natural boundaries. 
This preset is enabled by `regroup=True` (default). 
But there are other built-in [regrouping methods](#regrouping-methods) that allow you to customize the regrouping logic. 
This preset is just a predefined combination of those methods.

https://user-images.githubusercontent.com/28970749/226504985-3d087539-cfa4-46d1-8eb5-7083f235b429.mp4

```python
result0 = model.transcribe('audio.mp3', regroup=True) # regroup is True by default
# regroup=True is same as below
result1 = model.transcribe('audio.mp3', regroup=False)
(
    result1
    .split_by_punctuation([('.', ' '), '。', '?', '？', ',', '，'])
    .split_by_gap(.5)
    .merge_by_gap(.15, max_words=3)
    .split_by_punctuation([('.', ' '), '。', '?', '？'])
)
# result0 == result1
```
#### Regrouping Methods
- [split_by_gap()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L526-L543)
- [split_by_punctuation()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L579-L595)
- [split_by_length()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L637-L658)
- [merge_by_gap()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L547-L573)
- [merge_by_punctuation()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L599-L624)
- [merge_all_segments()](https://github.com/jianfch/stable-ts/blob/7c6953526dce5d9058b23e8d0c223272bf808be7/stable_whisper/result.py#L630-L633)

### Locating Words
You can locate words with regular expression.
```python
# Find every sentence that contains "and"
matches = result.find(r'[^.]+and[^.]+\.')
# print the first match if there is one
if matches:
  print(f'match: {matches[0].text_match}\n'
        f'text: {matches[0].text}\n'
        f'start: {matches[0].start}\n'
        f'end: {matches[0].end}\n')
  segments = matches[0].segments
  
# Find two words before and after "and" in the matches
matches = matches.find(r'\W\(w+)\Wand\W\(w+)\W')
if matches:
  print(f'match: {matches[0].text_match}\n'
        f'text: {matches[0].text}\n'
        f'start: {matches[0].start}\n'
        f'end: {matches[0].end}\n')
  segments = matches[0].segments
```
Parameters: 
[find()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/result.py#L768-L773)

### Boosting Performance
* One of the methods that Stable-ts uses to increase timestamp accuracy 
and reduce hallucinations is silence suppression, enabled by `suppress_silence=True` (default).
This method essentially suppresses the timestamps where the audio 
by suppressing the corresponding tokens during inference and also readjusting the timestamps after inference. 
To figure out which parts of the audio track are silent or contain no speech, Stable-ts supports non-VAD and VAD methods.
The default is `vad=False`. The VAD option uses [Silero VAD](https://github.com/snakers4/silero-vad) (requires PyTorch 1.12.0+). 
See [Visualizing Suppression](#visualizing-suppression).
* The other method, enabled by `demucs=True`, uses [Demucs](https://github.com/facebookresearch/demucs)
to isolate speech from the rest of the audio track. Generally best used in conjunction with silence suppression.
Although Demucs is for music, it is also effective at isolating speech even if the track contains no music.

### Visualizing Suppression
You can visualize which parts of the audio will likely be suppressed (i.e. marked as silent). 
Requires: [Pillow](https://github.com/python-pillow/Pillow) or [opencv-python](https://github.com/opencv/opencv-python).

#### Without VAD
```python
import stable_whisper
# regions on the waveform colored red are where it will likely be suppressed and marked as silent
# [q_levels]=20 and [k_size]=5 (default)
stable_whisper.visualize_suppression('audio.mp3', 'image.png', q_levels=20, k_size = 5) 
```
![novad](https://user-images.githubusercontent.com/28970749/225825408-aca63dbf-9571-40be-b399-1259d98f93be.png)

#### With [Silero VAD](https://github.com/snakers4/silero-vad)
```python
# [vad_threshold]=0.35 (default)
stable_whisper.visualize_suppression('audio.mp3', 'image.png', vad=True, vad_threshold=0.35)
```
![vad](https://user-images.githubusercontent.com/28970749/225825446-980924a5-7485-41e1-b0d9-c9b069d605f2.png)
Parameters: 
[visualize_suppression()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/stabilization.py#L334-L355)

### Encode Comparison 
You can encode videos similar to the ones in the doc for comparing transcriptions of the same audio. 
```python
import stable_whisper

stable_whisper.encode_video_comparison(
    'audio.mp3', 
    ['audio_sub1.srt', 'audio_sub2.srt'], 
    output_videopath='audio.mp4', 
    labels=['Example 1', 'Example 2']
)
```
Parameters: 
[encode_video_comparison()](https://github.com/jianfch/stable-ts/blob/d30d0d1cfb5b17b4bf59c3fafcbbd21e37598ab9/stable_whisper/video_output.py#L10-L27)

### Tips
- for reliable segment timestamps, do not disable word timestamps with `word_timestamps=False` because word timestamps are also used to correct segment timestamps
- use `demucs=True` and `vad=True` for music but also works for non-music
- if audio is not transcribing properly compared to whisper, try `mel_first=True` at the cost of more memory usage for long audio tracks
- enable dynamic quantization to decrease memory usage for inference on CPU (also increases inference speed for large model);
`--dq true`/`dq=True` for `stable_whisper.load_model`

#### Multiple Files with CLI 
Transcribe multiple audio files then process the results directly into SRT files.
```commandline
stable-ts audio1.mp3 audio2.mp3 audio3.mp3 -o audio1.srt audio2.srt audio3.srt
```

## Quick 1.X → 2.X Guide
### What's new in 2.0.0?
- updated to use Whisper's more reliable word-level timestamps method. 
- the more reliable word timestamps allow regrouping all words into segments with more natural boundaries.
- can now suppress silence with [Silero VAD](https://github.com/snakers4/silero-vad) (requires PyTorch 1.12.0+)
- non-VAD silence suppression is also more robust
### Usage changes
- `results_to_sentence_srt(result, 'audio.srt')` → `result.to_srt_vtt('audio.srt', word_level=False)` 
- `results_to_word_srt(result, 'audio.srt')` → `result.to_srt_vtt('output.srt', segment_level=False)`
- `results_to_sentence_word_ass(result, 'audio.srt')` → `result.to_ass('output.ass')`
- there's no need to stabilize segments after inference because they're already stabilized during inference
- `transcribe()` returns a `WhisperResult` object which can be converted to `dict` with `.to_dict()`. e.g `result.to_dict()`

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

## Acknowledgments
Includes slight modification of the original work: [Whisper](https://github.com/openai/whisper)
