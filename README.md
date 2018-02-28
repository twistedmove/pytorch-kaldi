# pytorch-kaldi


## Introduction:
This project releases a collection of codes and utilities to develop state-of-the-art DNN/RNN hybrid speech recognition systems. The DNN/RNN part is implemented in pytorch, while feature extraction, alignments, and decoding are performed with the Kaldi toolkit.  The current version of the provided system has the following features:
- different types of NNs (e.g., MLP, RNN, LSTM, GRU, Minimal GRU, Light GRU)
- recurrent dropout
- batch and layer normalization
- unidirectional/bidirectional RNNs
- residual/skip connections
- twin regularization
- python2/python3 compatibility
- multi-gpu training
- recovery/saving checkpoints
- easy interface with kaldi.

The provided solution is designed for large-scale speech recognition experiments on both standard machines and HPC clusters. 

## Prerequisites:
Linux is required (we tested our release on ubuntu 17.04 and various versions of Debian).

We recommend to run the codes on a GPU machine. Make sure that the cuda libraries (https://developer.nvidia.com/cuda-downloads) are installed and correctly working. We tested our system on cuda 9.0,9.1 and 8.0.
Make sure that python is installed (the code is tested with python 2.7 and python 3.6). Even though not mandatory, we suggest to use Anaconda (https://anaconda.org/anaconda/python).

If not already done, install pytorch (http://pytorch.org/). We tested our codes on pytorch 0.3.0 and pytorch 0.3.1. Older version of pytorch are likely to rise errors. To check your installation, type “python” and, once entered into the console, type “import torch”. Make sure everything is fine.

If not already done, install Kaldi (http://kaldi-asr.org/). As suggested during the installation, do not forget to add the path of the Kaldi binaries into $HOME/.bashrc. As a first test to check the installation, open a bash shell, type “copy-feats” and make sure no errors appear.

Install kaldi-io package from the kaldi-io-for-python project (https://github.com/vesis84/kaldi-io-for-python). It provides a simple interface between kaldi and python. To install it:
run git clone https://github.com/vesis84/kaldi-io-for-python.git
add PYTHONPATH=${PYTHONPATH}: to $HOME/.bashrc
now type import kaldi_io from the python console and make sure the package is correctly imported. You can find more info (including some reading and writing tests) on https://github.com/vesis84/kaldi-io-for-python
The implementation of the RNN models orders the training sentences according to their length. This allows the system to minimize the need of zero padding when forming minibatches. The duration of each sentence is extracted using sox. Please, make sure it is installed (it is only used when generating the feature lists in the create_chunk.sh)


## How to run a TIMIT experiment:
Even though the code can be easily adapted to any speech dataset, in the following part of the documentation we provide an example based on the popular TIMIT dataset.
1. Run the Kaldi s5 baseline of TIMIT.
This step is necessary to compute features and labels later used to train the pytorch MLP. In particular:
go to $KALDI_ROOT/egs/timit/s5.
run the script run.sh. Make sure everything works fine. Please, also run the Karel’s DNN baseline using local/nnet/run_dnn.sh.
Compute the alignments for test and dev data with the following commands.
If you wanna use tri3 alignments, type:
steps/align_fmllr.sh --nj 4 data/dev data/lang exp/tri3 exp/tri3_ali_dev

steps/align_fmllr.sh --nj 4 data/test data/lang exp/tri3 exp/tri3_ali_test


If you wanna use dnn alignments (as suggested), type:
steps/nnet/align.sh --nj 4 data-fmllr-tri3/dev data/lang exp/dnn4_pretrain-dbn_dnn exp/dnn4_pretrain-dbn_dnn_ali_dev

steps/nnet/align.sh --nj 4 data-fmllr-tri3/test data/lang exp/dnn4_pretrain-dbn_dnn exp/dnn4_pretrain-dbn_dnn_ali_test


2. Split the feature lists into chunks.
Go to the pytorch-kaldi folder. The create_chunks.sh script first shuffles or sorts (based on the sentence length) a kaldi feature list and then split it into a certain number of chunks. Shuffling a list could be good for feed-forward DNNs, while a sorted list can be useful for RNNs (not used here). The code also computes per-speaker and per-sentence CMVN.
For mfcc features run:
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/train mfcc_shu 5 train 0
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/dev mfcc_shu 1 dev 0
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/test mfcc_shu 1 test 0


For ordering the features according to their length (suggested for RNN models) run:
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/train mfcc_ord 5 train 1
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/dev mfcc_ord 1 dev 1
./create_chunks.sh $KALDI_ROOT/egs/timit/s5/data/test mfcc_ord 1 test 1



3. Setup the Config file.
Go into the cfg folder and open a config file (e.g,TIMIT_MLP.cfg,TIMIT_GRU.cfg) and modify them according to your paths.
tr_fea_scp contains the list of features created with create_chunks.sh.
tr_fea_opts allows users to easily add normalizations, derivatives and other types of feature processing.
tr_lab_folder is the kaldi folder containing the alignments (labels).
tr_lab_opts allows users to derive context-dependent phone targets (when set to ali-to-pdf) or monophone targets (when set to ali-to-phones --per-frame).
Modify the paths for dev and test data.
Feel free to modify the DNN architecture and the other optimization parameters according to your needs.
The required count_file (used to normalize the DNN posteriors before feeding the decoder and automatically created by kaldi when running s5 recipe) can be found here: $KALDI_ROOT/egs/timit/s5/exp/dnn4_pretrain-dbn_dnn/ali_train_pdf.counts.
Use the option use_cuda=1 for running the code on a GPU (strongly suggested).
Use the option save_gpumem=0 to save gpu memory. The code will be a little bit slower (about 10-15%), but it saves gpu memory. Use save_gpumem=1 only if your GPU has more that 2GB of memory.
4. Train the DNN.
Type the following command to run DNN training :
./run_exp.sh cfg/TIMIT_MLP.cfg > log.log 
or 
./run_exp.sh cfg/TIMIT_GRU.cfg > log.log 


Note that run_exp performs a full ASR experiment (training,forward and decoding).
If everything is working fine,  in the output folder specified in the cfg you should find:
a file “res.res” summarizing the training and eval performance over the various epochs. 
For the MFCC experiment you should obtain something like this:
For the GRU experiment you should obtain something like this

a folder “decode_test” containing the speech recognition results. If you type ./RESULTS you should be able to see the Phone Error Rate (PER%) for each experiment. For instance:
mfcc features: PER=18.0%
fMLLR features: PER=16.8%
The model *.pkl is the final model used for speech decoding
The files *.info reports loss and error performance for each training chunk
The file log.log contains possible errors occurred in the training procedure.
