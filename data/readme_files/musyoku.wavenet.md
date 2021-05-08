## WaveNet: A Generative Model for Raw Audio

This is the Chainer implementation of [WaveNet](http://arxiv.org/abs/1609.03499)

[この記事](http://musyoku.github.io/2016/09/18/wavenet-a-generative-model-for-raw-audio/)で実装したコードです。

まだ完成していませんが音声の生成はできます。

#### Todo:
- [x] Generating audio
- [ ] Local conditioning
- [ ] Global conditioning
- [ ] Training on CSTR VCTK Corpus

## Training the network

### Requirements

- Chainer 2
- scipy.io.wavfile

### Preprocessing

Donwsample your .wav to 16KHz / 8KHz to speed up convergence.

- [Aaudacity](http://www.audacityteam.org/)

### Create data directory

Add all .wav files to `/train_audio/wav`

### Hyperparameters

You can edit the hyperparameters of the network in `model.py` before running `train.py`, or edit `/params/params.json` after training starts.

### Training

run `train.py`

### Generating audio

run `generate.py`

Passing `--use_faster_wavenet` will generate audio faster than original WaveNet.

#### Listen to a sample generated by WaveNet

[🎶 music](https://drive.google.com/file/d/0ByQaxyG1S5JRWUZrQkpaMTJRNFk/view)

## Implementation

![figure](https://github.com/musyoku/musyoku.github.io/blob/master/images/post/2016-09-17/arch.png?raw=true)

![figure](https://github.com/musyoku/musyoku.github.io/blob/master/images/post/2016-09-17/block.png?raw=true)

![figure](https://github.com/musyoku/musyoku.github.io/blob/master/images/post/2016-09-17/actual_data.png?raw=true)