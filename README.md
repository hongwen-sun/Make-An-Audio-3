# Make-An-Audio 3: Transforming Text into Audio via Flow-based Large Diffusion Transformers

PyTorch Implementation of [Lumina-t2x](https://arxiv.org/abs/2405.05945)

We will provide our implementation and pre-trained models as open-source in this repository recently.

[![arXiv](https://img.shields.io/badge/arXiv-Paper-<COLOR>.svg)](https://arxiv.org/abs/2305.18474)
[![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-blue)](https://huggingface.co/spaces/AIGC-Audio/Make-An-Audio-3)
[![GitHub Stars](https://img.shields.io/github/stars/Text-to-Audio/Make-An-Audio-3?style=social)](https://github.com/Text-to-Audio/Make-An-Audio-3)

## Use a pre-trained model
We provide our implementation and pre-trained models as open-source in this repository.

Visit our [demo page](https://make-an-audio-2.github.io/) for audio samples.

## News
- June, 2024: **[Make-An-Audio-3 (Lumina-Next)](https://arxiv.org/abs/2405.05945)** released in [Github](https://github.com/Text-to-Audio/Make-An-Audio-3).

[//]: # (- May, 2024: **[Make-An-Audio-2]&#40;https://arxiv.org/abs/2207.06389&#41;** released in [Github]&#40;https://github.com/bytedance/Make-An-Audio-2&#41;.)
[//]: # (- August, 2023: **[Make-An-Audio]&#40;https://arxiv.org/abs/2301.12661&#41; &#40;ICML 2022&#41;** released in [Github]&#40;https://github.com/Text-to-Audio/Make-An-Audio&#41;. )

## Install dependencies

Note: You may want to adjust the CUDA version [according to your driver version](https://docs.nvidia.com/deploy/cuda-compatibility/#default-to-minor-version).

```bash
conda create -n Make_An_Audio_3 -y
conda activate Make_An_Audio_3
conda install python=3.11 pytorch==2.1.0 torchvision==0.16.0 torchaudio==2.1.0 pytorch-cuda=12.1 -c pytorch -c nvidia -y
pip install -r requirements.txt
pip install flash-attn --no-build-isolation
Install [nvidia apex](https://github.com/nvidia/apex) (optional)
```

## Quick Started
### Pretrained Models

Simply download the 500M weights from [![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-blue)](https://huggingface.co/spaces/AIGC-Audio/Make-An-Audio-3/tree/main/useful_ckpts)

 Model     | Pretraining Data   |  Path  
|-----------|--------------------|--------------------------------------------------------------------------------
| M (160M)  | AudioCaption       |[Here](https://huggingface.co/spaces/AIGC-Audio/Make-An-Audio-3/tree/main/useful_ckpts)
| L (520M)  | AudioCaption       |[TBD]
| XL (750M) | AudioCaption       |[TBD]
| 3B        | AudioCaption       |[TBD]
    
### Generate audio/music from text
```
python3 scripts/txt2audio_for_2cap_flow.py 
--outdir output_dir -r  checkpoints_last.ckpt  -b configs/txt2audio-cfm1-cfg-LargeDiT3.yaml --scale 3.0 
--vocoder-ckpt useful_ckpts/bigvnat --test-dataset audiocaps 
```

### Generate audio/music from audiocaps or musiccaps test dataset
- remember to relatively change `config["test_dataset]`
```
python3 scripts/txt2audio_for_2cap_flow.py 
--outdir output_dir -r  checkpoints_last.ckpt  -b configs/txt2audio-cfm1-cfg-LargeDiT3.yaml --scale 3.0 
--vocoder-ckpt useful_ckpts/bigvnat --test-dataset testset
```

### Generate audio/music from video
```
python3 scripts/video2audio_flow.py 
--outdir output_dir -r  checkpoints_last.ckpt  -b configs/video2audio-cfm1-cfg-LargeDiT1-moe.yaml --scale 3.0 
--vocoder-ckpt useful_ckpts/bigvnat --test-dataset vggsound 
```

## Train
### Data Preparation
- We can't provide the dataset download link for copyright issues. We provide the process code to generate melspec, count audio duration, and generate structured captions.  
- Before training, we need to construct the dataset information into a tsv file, which includes name (id for each audio), dataset (which dataset the audio belongs to), audio_path (the path of .wav file),caption (the caption of the audio) ,mel_path (the processed melspec file path of each audio), duration (the duration of the audio). We provide a tsv file of audiocaps test set: audiocaps_test_struct.tsv as a sample.
- We provide a tsv file of the audiocaps test set: ./audiocaps_test_16000_struct.tsv as a sample.

### Generate the melspec file of audio
Assume you have already got a tsv file to link each caption to its audio_path, which means the tsv_file have "name","audio_path","dataset" and "caption" columns in it.
To get the melspec of audio, run the following command, which will save mels in ./processed
```
python preprocess/mel_spec.py --tsv_path tmp.tsv --num_gpus 1 --max_duration 10
```

### Count audio duration
To count the duration of the audio and save duration information in tsv file, run the following command: 
```
python preprocess/add_duration.py --tsv_path tmp.tsv
```

### Generated structure caption from the original natural language caption
Firstly you need to get an authorization token in openai(https://openai.com/blog/openai-api), here is a tutorial(https://www.maisieai.com/help/how-to-get-an-openai-api-key-for-chatgpt). Then replace your key of variable openai_key in preprocess/n2s_by_openai.py. Run the following command to add structed caption, the tsv file with structured caption will be saved into {tsv_file_name}_struct.tsv:
```
python preprocess/n2s_by_openai.py --tsv_path tmp.tsv
```

### Place Tsv files
After generating the structure caption, put the tsv with the structed caption to ./data/main_spec_dir . And put tsv files without structured caption to ./data/no_struct_dir

Modify the config data.params.main_spec_dir and data.params.main_spec_dir.other_spec_dir_path respectively in config file configs/text2audio-ConcatDiT-ae1dnat_Skl20d2_struct2MLPanylen.yaml .

## Train variational autoencoder
Assume we have processed several datasets, and save the .tsv files in tsv_dir/*.tsv . Replace data.params.spec_dir_path with tsv_dir in the config file. Then we can train VAE with the following command. If you don't have 8 GPUs in your machine, you can replace --GPUs 0,1,...,gpu_nums
```
python main.py --base configs/research/autoencoder/autoencoder1d_kl20_natbig_r1_down2_disc2.yaml -t --gpus 0,1,2,3,4,5,6,7
```

## Train latent diffsuion
After trainning VAE, replace model.params.first_stage_config.params.ckpt_path with your trained VAE checkpoint path in the config file.
Run the following command to train Diffusion model
```
python main.py --base configs/research/text2audio/text2audio-ConcatDiT-ae1dnat_Skl20d2_freezeFlananylen_drop.yaml -t  --gpus 0,1,2,3,4,5,6,7
```

## Evaluation
Please refer to [Make-An-Audio](https://github.com/Text-to-Audio/Make-An-Audio?tab=readme-ov-file#evaluation).


## Acknowledgements
This implementation uses parts of the code from the following Github repos:
[Make-An-Audio](https://github.com/Text-to-Audio/Make-An-Audio),
[AudioLCM](https://github.com/Text-to-Audio/AudioLCM),
[CLAP](https://github.com/LAION-AI/CLAP),
as described in our code.



## Citations ##
If you find this code useful in your research, please consider citing:
```bibtex
```

# Disclaimer ##
Any organization or individual is prohibited from using any technology mentioned in this paper to generate someone's speech without his/her consent, including but not limited to government leaders, political figures, and celebrities. If you do not comply with this item, you could be in violation of copyright laws.
