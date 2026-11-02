# Video Quality Inference

Can you tell what resolution a Netflix stream is playing just by looking at its network traffic? This project builds a pipeline that does that. It reads a packet capture, turns traffic into features, and trains a model that predicts the stream's video resolution (240p, 360p, 480p, 720p or 1080p) without ever seeing the video.

## What the notebook does

**Part 1: from packets to features**
- Reads `netflix.pcap` (141,471 packets) and uses the DNS answers to find the 15 Netflix server IPs, which cover 138,633 packets.
- Splits the traffic into 10-second windows and builds 7 features per window: download and upload packet counts and bytes, average and max download packet size, and the average gap between download packets.
- Adds a segment-request count per window by counting large upstream packets sent to Netflix's video servers (208 requests in total).
- The capture was truncated to 200 bytes per packet, so sizes use each packet's original wire length.

**Part 2: training a model**
- Loads a labeled dataset of 52,279 windows (251 columns) and keeps the 49,748 rows with a valid resolution.
- Drops columns that would let the model cheat or memorize (values measured by the video player, IDs, timestamps) and columns with a single value, leaving 212 features.
- Splits train and test by viewing session, so windows from one session never land on both sides.
- Trains a random forest and tunes it with a grid search.
- On the held-out sessions: **86.6% accuracy, 0.79 macro F1, 0.965 ROC AUC** (one-vs-rest), with a confusion matrix and ROC curves in the notebook.

**Part 3: a real session**
- Runs the trained model on one real Netflix session (59 ten-second windows) and plots predicted against true resolution over time.

## Results and limits

The model is good on steady streams and weak on this real session: it gets about **34%** of the 59 windows right. The true resolution in that session changes 27 times, far more than most sessions in the training data, and many features summarize everything since the session started, so they react slowly to sudden drops. Resolutions that are rare in the data (240p) are also the hardest to catch. I left this result in because it is the honest number.

## Running it

1. Install the packages: `pip install pandas matplotlib scapy scikit-learn jupyter`
2. Download the data into a `data/` folder next to the notebook:

```
mkdir data && cd data
BASE=https://raw.githubusercontent.com/noise-courses/data/main/video-qoe
curl -O $BASE/netflix.pcap
curl -O $BASE/netflix_session.pkl
curl -O $BASE/netflix_dataset.pkl.xz
xz -d netflix_dataset.pkl.xz
cd ..
```

3. Open the notebook and run all cells. The tuning cell takes a few minutes.

The data comes from the course data repository, [noise-courses/data](https://github.com/noise-courses/data), and is not included here.
