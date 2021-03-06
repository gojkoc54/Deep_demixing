# Deep_demixing
Official repository of the paper 'Deep Demixing: Reconstructing the evolution of epidemics using graph neural networks' \
https://arxiv.org/abs/2011.09583


## 0. Prerequisites

- [anaconda](https://docs.anaconda.com/anaconda/install/) (ver=4.8.3)
- python (3.7)
- [pytorch](https://pytorch.org/get-started/locally/) (1.6.0)
- [pytorch_geometric ](https://pytorch-geometric.readthedocs.io/en/latest/notes/installation.html)(1.6.1)

To make it less tedious for you, we provide a conda env file `./environment.yml`. With anaconda installed, direct to the project root dir and simply run the following lines in your terminal should set everything except for torch-geometric.

```
git clone https://github.com/gojkoc54/Deep_demixing.git
cd Deep_demixing
conda env create -f environment.yml
source activate torch
```

Next, in this conda environment, install torch-geometric and its dependencies,
```
pip install torch-scatter==latest+cu102 -f https://pytorch-geometric.com/whl/torch-1.6.0.html
pip install torch-sparse==latest+cu102 -f https://pytorch-geometric.com/whl/torch-1.6.0.html
pip install torch-cluster==latest+cu102 -f https://pytorch-geometric.com/whl/torch-1.6.0.html
pip install torch-spline-conv==latest+cu102 -f https://pytorch-geometric.com/whl/torch-1.6.0.html
pip install torch-geometric
```

Note: in commands above, please configure "cu102" and "torch-1.6.0" to make sure they match CUDA version {cpu, cu92, cu101, cu102} and torch version {1.4.0, 1.5.0, 1.6.0} on your machine. You find out the versions by
```
(torch)$ python -c "import torch; print(''.join(torch.version.cuda.split('.')),end=' ; '); print('torch-'+torch.__version__);"
>>> 102 ; torch-1.6.0
```

## 1. Getting Started

The organization of data and code in this repo is as follows,

```
Deep_demixing/
        |-- datasets/
        |   |-- exp_1/
        |   |   |-- 5_steps/
        |   |   |   |-- RG_100nodes_20steps_1000sims.obj
        |   |   |   `-- RG_100nodes_20steps_5000sims.obj
        |   |   |-- 10_steps/
        |   |   |-- 15_steps/
        |   |   `-- 20_steps/
        |   |-- exp_2/
        |   `-- ...
        |-- scripts/
        |   |-- exp0-minimal.py
        |   |-- exp1-num_steps.py
        |   `-- ...
        |-- models.py
        |-- trainer.py
        `-- utils.py
```


### 1) Make Sure Basic Training Works
To test a minimal example, run in the commandline,
```
(torch)$ python scripts/exp0-minimal.py
```
(After resolving any extra package requirements,) this will use the toy dataset that we provide you in `./datasets/` and train our DDmix model (we call it CVAE_UNET here) and 3 baseline models (MLP, CNN_time, and CNN_nodes) from scratch. As the training starts, a new folder named `./models/` should be created. Checkpoint files and the final models will be saved under this directory. You can monitor the training with [tensorboard](https://pytorch.org/docs/stable/tensorboard.html) (tb). Basically, you run
```
(torch)$ tensorboard --logdir=MODEL_LOGS_DIR --port=PORT_NUM --bind_all
```
in which `MODEL_LOGS_DIR` is the directory that contains your training logs (e.g. `./models/exp_0/`); the default `PORT_NUM`=6006; the `--bind_all` flag can be dropped if you are on your local machine.

Then in your browser, go to `localhost:PORT_NUM` (if local) or `REMOTE_IP_ADDR:PORT_NUM` (if remote) will take you to the tb dashboard. After the training finishes, the program will automatically stop. In this minimal working example, max epoch is 10 without auto stop for simplicity.

As you may have already found out, the entry point is in `./scripts/*.py`. You can modify configurations and hyperparameters (e.g. number of epochs, minibatch size, frequency of saving checkpoint models, learning rate, regularization coefficient, whether to use gpu and whether to do parallel training, ... etc.) in these files to train models in your favorite flavor.


### 2) Experiments
Data under directories `./datasets/exp_NUM/` are available for running the following experiments. We provide training scripts for experiments to reproduce our reported results. The scripts are given as `./scripts/exp_NUM-DESC.py`. Specifically, we have
* `exp1-num_steps`: Train models with data on a 100-node graph and various number of time-steps in {5,10,15,20};
* `exp8-less_data`: Train models using various amount of training samples: number of train samples in {0,4,8,16, 32, 50, 100,250, 500,1000,2000,4500};
* `exp10-real_graph`: Train models using synthetic epidemics on a real temporal contact graph.

Similar to the minimal example above, all these scripts can be run by
```
(torch)$ python ./script/exp_NUM-DESC.py
```

## 2. Have More Fun

So far so good? Now it's time to dig into the code and see what exciting things you can try out with our framework!

First of all, the data -- Our experiments rely on synthetic SIRS epidemics on (synthetic or real-world) graphs. This brief tutorial will start with a guidance for you to generate your own dataset. Next, we will show where and how our models are declared. We will also walk you through the `Trainer` class, a wrapper that we designed to manage the train, evaluate and tune models in a systematic manner. Finally, we will show some basic ways to visualize the results and compare performance.

#### i. Data Generation
In addition to our readily prepared datasets, we also include a script in this repo ([`./SIRS_outbreak_real.ipynb`](./SIRS_outbreak_real.ipynb)) to walk you through the process of generating epidemic data from scratch. Basically, it loads real contact graphs and spread synthetic epidemics on them, yet the "synthetic epidemics on synthetic graph" data generation follows exactly the same principles except that the graph itself will also be a synthetic RG graph.
> For more details, please refer to our data script [`./SIRS_outbreak_real.ipynb`](./SIRS_outbreak_real.ipynb).


#### ii. Model and Trainer Declaration

All the candidate models are declared in `./models.py`. They inherit from the `torch.nn.Module` class, and you can find its documentation [here](https://pytorch.org/docs/stable/generated/torch.nn.Module.html) and a detailed tutorial to customize it [here](https://pytorch.org/tutorials/beginner/pytorch_with_examples.html#pytorch-custom-nn-modules). Depending on whether the model is graph-based, you may choose from the 2 types of trainers defined in `./trainer.py`, namely the `CVAE_Pr_Trainer` and the `Benchmark_Trainer`. Here we introduce the usage of `CVAE_Pr_Trainer` in detail; the other one is very much similar or even simpler.

----
- CLASS trainer.GCVAE_Trainer(model, check_path, optimizer, resume=True, **kwargs)</span> [[SOURCE]](./trainer.py#L293)
  - A wrapper for PyTorch models.
  - Parameters:
    - **model** (*torch.nn.Module*) – the PyTorch model to train/validate/tune.
    - **check_path** (*string*) - the path to save/load the model.
    - **optimizer** (*torch.optim.Optimizer*) - the PyTorch optimizer to use for training the model.
    - **resume** (*bool*) - if True, load model parameters from a previous checkpoint if available. Otherwise, or if no checkpoint is found, train the model from scratch.
    - **kwargs** (*dict*) - a dict containing optional values of training options.
        - **display_step** (*int*, *optional*) - how many steps to display or log training stats.
        - **gradient_clipping** (*float*, *optional*) - parameter for gradient clipping.
        - **decay_step** ( [*int*, *float*], *optional*) - number of steps ( decay_step[0] ) before the learning rate decays by a factor ( decay_step[1] ).
        - **l2** (*float*, *optional*) - if set, this will be the coefficient for l2 regularization.
        - **max_steps** (*int*, *optional*) - if set, this will be the number of time-steps that we consider when training the model. For example, if we pass in input data `y` of shape (BN, T) and max_steps=T_m ( <T ), then you can think of it as equivalent to slicing `y` into `y[:,:T_m]`.
        - **auto_stop** (*int*, *optional*) - during training, if the validation loss stops dropping for no more than this number of epochs, the training has to hang in there. Once exceeding this number, the trainer will automatically terminate.
  - Methods:
    - **train** (*epoch: int, criterion: dict, loader: DataLoader*) → None
      - Parameters:
        - **epoch** (*int*) - current epoch number.
        - **criterion** (*dict*) - a dict of loss functions to use. Should be in the format of `{'kld': ..., 'bce': ..., 'rule': ...}`. Among the fixed keys, `'kld'` is the KL-divergence loss between posterior and prior nets; `'bce'` is the binary cross entropy loss of the reconstructed graph signals; `'rule'` is the spreading rule based regularizer that's specific to our definition. (Our rule basically says if your neighbors are not infected in day t-1, then you cannot get infected in day t.)
        - **loader** (*torch_geometric.data.DataLoader/DataListLoader*) - training dataloader.
    - **predict** (*epoch: int, criterion: dict = None, loader:DataLoader = None, save: bool = False*) → y_gt, y_pr, A_eidx
      - Parameters:
        - **epoch** (*int*) - current epoch number.
        - **criterion** (*dict*, *optional*) - same as in `.train()`.
        - **loader** (*torch_geometric.data.DataLoader/DataListLoader*) - validation or test dataloader.
        - **save** (*bool*, *optional*) - if True, save the model if the predicted loss turns out to be the current smallest.
      - Returns:
        - **y_gt** (*numpy.array*) - ground truth graph signals.
        - **y_pr** (*numpy.array*) - predicted probabilities of reconstructed graph signals.
        - **A_eidx** (*numpy.array*) - the associated graph adjacency in the form of edge index.
----

> For more details and extensive usage, please refer to the training scripts [`./scripts/*.py`](./scripts/).

#### iii. Analysis of Results
We analyze and compare results yielded by candidate model both quantitatively and qualitatively. Quantitatively, we adopt the mean square error (MSE) to evaluate the node-level binary (infected / not infected) reconstruction performance. Additionally, we examine the top-k accuracy of sourcing the initial infected point performed by these models. Qualitatively, here we display the reconstructed graphs over time to show that our proposed model indeed does well finding the patient(s) "zero".

<br>

**Table 1**: MSE of predictions on unseen graphs with different graphdensity for two different lengths of the epidemics.
| Graph density  | Baseline       | Denser        | Sparser        |
|----------------|:--------------:|:-------------:|:--------------:|
| **Time steps (T)** | **10** \| **20** | **10** \| **20**| **10** \| **20**|
| MLP            | .217  \|  .233 | .289  \| .268 | .254   \| .254 |
| CNN-nodes      | .106  \|  .188 | .270  \| .239 | .177   \| .232 |
| CNN-time       | .143  \|  .194 | .267  \| .240 | .168   \| .226 |
| DDmix    |**.101** \| **.177** | **.160** \| **.197** | **.075** \| **.216** |

<br>

**Table 2**: Real network experiment. Accuracy (%) of identifying the source of epidemics (20 steps, 10 classes).
| Models     | top-1       | top-3       | top-5         |
|------------|:-----------:|:-----------:|:-------------:|
| CNN-time   | 11.6        | 30.4        | 53.4          |
| CNN-nodes  | 12.8        | 36.8        | 60.0          |
| DDmix      |**25.9**     | **61.2**    | **82.9**      |

<br>

![graph](./figs/vis.png)
**Figure 1**. DDmix is able to trace the cluster of initially infected nodes to its accurate source, whereas the non-graph-aware methods cannot. Unlike DDmix's reconstruction where one can clearly observe local and global spreading paths, their node probabilities just universally fade away as going back in time.

> For more details, please refer to our visualization script [`./results_vis.ipynb`](./results_vis.ipynb).


## References
- http://www.sociopatterns.org/datasets/primary-school-temporal-network-data/
