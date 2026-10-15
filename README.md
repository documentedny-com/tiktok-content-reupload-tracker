# TikTok Content Reupload Tracker

This repository contains a TikTok re-upload analysis pipeline. Given a list of TikTok video URLs, the pipeline downloads the videos, transcribes their audio, translates any non-English transcripts to English, and then clusters the transcripts to identify groups of videos that share the same (or near-identical) spoken content — a signal that the same clip has been re-uploaded across multiple accounts or at different times.

An optional pre-step is included to convert TikTok account export data (such as a particular account's watch history) into a CSV that can be used as the input for this pipeline.

## Input data

The main pipeline reads from a CSV in `raw/` where each row represents one TikTok video and includes, at minimum, a `link` column with a full TikTok URL. See [`raw/sample_video_list.csv`](raw/sample_video_list.csv) as an example.

## Pipeline overview

The notebooks run in order:

### `00` (Optional) — Convert watch history JSON to CSV

[`notebooks/00-watch-history-to-csv.ipynb`](notebooks/00-watch-history-to-csv.ipynb)

Converts TikTok account export JSON into a flat CSV using snake_case headers. It reads `Your Activity -> Watch History -> VideoList` (or `Video List`) and writes `raw/watch_history.csv`.

If you use this output as input for Step 01, make sure to change the name of `ORIGINAL_VIDEO_LIST_FILE`.

### `01` — Download videos

[`notebooks/01-scrape.ipynb`](notebooks/01-scrape.ipynb)

Reads the metadata CSV and filters to the months you specify, then downloads each video as an mp4 file using [yt-dlp](https://github.com/yt-dlp/yt-dlp). An archive file tracks which videos have already been downloaded so re-running the notebook never downloads the same video twice.

### `02` — Transcribe audio

[`notebooks/02-transcribe.ipynb`](notebooks/02-transcribe.ipynb)

Scans the video folder for mp4/mov/mkv/webm/mp3/wav/m4a/aac files and transcribes each one using [faster-whisper](https://github.com/guillaumekln/faster-whisper) (an optimised implementation of OpenAI Whisper). Audio is extracted to a cache folder so repeated runs skip re-extraction. Transcription runs in parallel threads and checkpoints results to CSV every `SAVE_EVERY` files so progress is not lost if the process is interrupted. The detected language of each video is saved alongside the transcript.

### `03` — Translate transcriptions

[`notebooks/03-translate.ipynb`](notebooks/03-translate.ipynb)

Reads the transcriptions CSV and translates any non-English transcripts to English using Google Translate (via the `deep-translator` library). English transcripts are passed through unchanged. The notebook retries failed translations once and saves progress every `SAVE_EVERY` rows. After translation, the transcripts are merged with the original video metadata and saved as a combined CSV.

### `04` — Cluster by content similarity

[`notebooks/04-cluster.ipynb`](notebooks/04-cluster.ipynb)

Groups videos whose transcripts contain the same (or very similar) spoken content. Two similarity signals are combined:

- **Semantic similarity** — sentence embeddings from [`all-MiniLM-L6-v2`](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) are compared with cosine similarity to catch paraphrased or near-identical speech.
- **Containment similarity** — detects cases where one transcript is a verbatim substring of another (e.g. a clipped re-upload of a longer original video).

The combined distance matrix is fed into [DBSCAN](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.DBSCAN.html) so that clusters form automatically without needing to specify the number of groups in advance. Videos that don't match any cluster are labelled as noise (`cluster_id = -1`) and excluded from the output.

## Outputs

| File                                                                                                        | Description                                                                                 |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [`output/videofiles-extracted/`](sample_output/videofiles-extracted/)                                       | Downloaded mp4 video files, named by TikTok video ID                                        |
| [`output/videofiles-extracted/downloaded-list.txt`](sample_output/videofiles-extracted/downloaded-list.txt) | yt-dlp archive file used to prevent duplicate downloads                                     |
| [`output/videofiles-extracted/*.info.json`](sample_output/videofiles-extracted/)                            | Per-video yt-dlp metadata sidecar JSON files                                                |
| [`output/videofiles-extracted/video_metadata.csv`](sample_output/videofiles-extracted/video_metadata.csv)   | Consolidated metadata table from sidecar JSON files (author, date, engagement, etc.)        |
| [`output/audio-cache/`](sample_output/audio-cache/)                                                         | Extracted WAV audio files cached during transcription                                       |
| [`output/transcriptions.csv`](sample_output/transcriptions.csv)                                             | Raw transcription text and detected language for each video                                 |
| [`output/transcriptions_translated.csv`](sample_output/transcriptions_translated.csv)                       | Transcriptions with an added `transcript_en` column containing English translations         |
| [`output/transcriptions_and_translations.csv`](sample_output/transcriptions_and_translations.csv)           | Translations merged with original video metadata                                            |
| [`output/cluster_summary.csv`](sample_output/cluster_summary.csv)                                           | One row per cluster: cluster ID, number of videos in the cluster, and an example transcript |
| [`output/clustered_translations.csv`](sample_output/clustered_translations.csv)                             | Full video metadata with cluster assignments (noise videos excluded)                        |

## Running the pipeline

### Requirements

- Python 3.10+
- The Python libraries listed in [`requirements.txt`](requirements.txt)

### Setup

```bash
pip install -r requirements.txt
```

The transcription step also requires the `ffmpeg` system dependency. Follow the installation instructions in the [ffmpeg install guide](https://github.com/oop7/ffmpeg-install-guide).

Open the notebooks in order and run all cells:

1. Optional: [`notebooks/00-watch-history-to-csv.ipynb`](notebooks/00-watch-history-to-csv.ipynb)
2. [`notebooks/01-scrape.ipynb`](notebooks/01-scrape.ipynb)
3. [`notebooks/02-transcribe.ipynb`](notebooks/02-transcribe.ipynb)
4. [`notebooks/03-translate.ipynb`](notebooks/03-translate.ipynb)
5. [`notebooks/04-cluster.ipynb`](notebooks/04-cluster.ipynb)

Each notebook has a **configuration block** near the top — adjust the paths and parameters there before running.

For Google Colab, each notebook includes commented-out lines for mounting Google Drive and installing dependencies inline.

## Known Issues & Troubleshooting

1. **Scraping videos directly from TikTok can be temperamental**

Rate limiting, behavioral tracking, and network connectivity issues, among other things can interfere with the scraping process and leave you with missing extraction videos or, in some cases, just missing audio.

_The Fix:_ Our script keeps a record of each video you download so you can run the scraper in batches, reducing volatility. Running the script in [`notebooks/02-transcribe.ipynb`](notebooks/02-transcribe.ipynb) can help you spot videos with missing audio in case you want to try scraping those again, too. 

2. **Google Translator services may shut you out**
   
While running [`notebooks/03-translate.ipynb`](notebooks/03-translate.ipynb), the script may return a `HTTP 429 (Too Many Requests)` and `HTTP 403 (Forbidden)` error when trying to make requests to Google Translate. This can sometimes happen when Google's automated anti-abuse systems detect high-frequency automated queries.

_The Fix:_ The [deep-translator](https://github.com/nidhaloff/deep-translator) library to allow you to easily swap in a different translation service if Google Translate is not working properly or doesn’t meet your standards. 

## Contributing 

If you do something cool and impactful with our code or otherwise want to contribute to this project, let us know! Drop us a line at [sam.rabiyah@documentedny.com](mailto:sam.rabiyah@documentedny.com) and [lam.vo@documentedny.com](mailto:lam.vo@documentedny.com).

## Licensing

All code in this repository is available under the [MIT License](https://opensource.org/licenses/MIT). The data files in the sample_output/ directory are available under the [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0) license. All files in the raw/ directory are released into the public domain.
