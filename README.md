# PyTorch Template Project For GANs

This project template is adapted from [PyTorch Template](https://github.com/victoresque/pytorch-template) to support GANs training and evaluation.

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [PyTorch Template Project For GANs](#pytorch-template-project-for-gans)
  - [Requirements](#requirements)
  - [Features](#features)
  - [Folder Structure](#folder-structure)
  - [Usage](#usage)
    - [Config file format](#config-file-format)
    - [Using config files](#using-config-files)
    - [Resuming from checkpoints](#resuming-from-checkpoints)
    - [Using Multiple GPU](#using-multiple-gpu)
  - [Customization](#customization)
    - [Custom CLI options](#custom-cli-options)
    - [Additional logging](#additional-logging)
    - [Testing](#testing)
    - [Checkpoints](#checkpoints)
    - [Tensorboard Visualization](#tensorboard-visualization)
  - [Acknowledgements](#acknowledgements)

<!-- /code_chunk_output -->

## Requirements

- Python >= 3.5 (3.6 recommended)
- PyTorch >= 0.4 (1.2 recommended)
- tqdm (Optional for `eval.py`)
- tensorboard >= 1.14 (see [Tensorboard Visualization](#tensorboard-visualization))

## Features

- Clear folder structure which is suitable for many deep learning projects.
- `.json` config file support for convenient parameter tuning.
- Customizable command line options for more convenient parameter tuning.
- Checkpoint saving and resuming.
- Abstract base classes for faster development:
  - `BaseGANTrainer` handles checkpoint saving/resuming, training process logging, and more.
  - `BaseDataLoader` handles batch generation, data shuffling, and validation data splitting.
  - `BaseGAN` provides basic GAN model summary.

## Folder Structure

```
pytorch-template/
│
├── train.py - main script to start training
├── eval.py - evaluation of trained model
│
├── config.json - holds configuration for training
├── parse_config.py - class to handle config file and cli options
│
├── base/ - abstract base classes
│   ├── base_data_loader.py
│   ├── base_gan.py
│   └── base_gan_trainer.py
│
├── data_loader/ - anything about data loading goes here
│   └── data_loaders.py
│
├── data/ - default directory for storing input data
│
├── model/ - models, losses, and metrics
│   ├── models/
│   │   └── gan.py
│   ├── metric.py
│   └── loss.py
│
├── saved/
│   ├── models/ - trained models are saved here
│   └── log/ - default logdir for tensorboard and logging output
│
├── trainer/ - trainers
│   └── trainer.py
│
├── logger/ - module for tensorboard visualization and logging
│   ├── visualization.py
│   ├── logger.py
│   └── logger_config.json
│
└── utils/ - small utility functions
    ├── util.py
    └── ...
```

## Usage

The code in this repo is an MNIST example of the template.
Try `python train.py -c config.json` to run code.

### Config file format

Config files are in `.json` format:

```javascript
{
  "name": "Mnist_GAN",        // training session name
  "n_gpu": 1,                   // number of GPUs to use for training.

  "arch": {
    "type": "GAN",       // name of model architecture to train
    "args": {
        "latent_dim": 100
    }
  },
  "data_loader": {
    "type": "MnistDataLoader",         // selecting data loader
    "args":{
      "data_dir": "data/",             // dataset path
      "batch_size": 64,                // batch size
      "shuffle": true,                 // shuffle training data before splitting
      "num_workers": 2,                // number of cpu processes to be used for data loading
    }
  },
  "optimizer_G": {                     // optimizer for generator
    "type": "Adam",
    "args":{
      "lr": 0.001,                     // learning rate
      "weight_decay": 0,               // (optional) weight decay
      "amsgrad": true
    }
  },
  "optimizer_D": {                     // optimizer for discriminator
    "type": "Adam",
    "args":{
      "lr": 0.001,
      "weight_decay": 0,
      "amsgrad": true
    }
  },
  "loss": "adversarial_loss",          // loss
  "metrics": [],
  "lr_scheduler_G": {
    "type": "StepLR",                  // learning rate scheduler for generator
    "args":{
      "step_size": 50,
      "gamma": 0.1
    }
  },
  "lr_scheduler_D": {
    "type": "StepLR",                  // learning rate scheduler for discriminator
    "args":{
      "step_size": 50,
      "gamma": 0.1
    }
  },
  "trainer": {
    "epochs": 100,                     // number of training epochs
    "save_dir": "saved/",              // checkpoints are saved in save_dir/models/name
    "save_period": 1,                  // save checkpoints every save_period epochs
    "verbosity": 2,                    // 0: quiet, 1: per epoch, 2: full

    "tensorboard": true,               // enable tensorboard visualization
  },
  "eval": {
    "save_dir": "saved/generated",
    "n_sample": 1000,
    "batch_size": 8
  }
}
```

Add addional configurations if you need.

### Using config files

Modify the configurations in `.json` config files, then run:

```
python train.py --config config.json
```

### Resuming from checkpoints

You can resume from a previously saved checkpoint by:

```
python train.py --resume path/to/checkpoint
```

### Using Multiple GPU

You can enable multi-GPU training by setting `n_gpu` argument of the config file to larger number.
If configured to use smaller number of gpu than available, first n devices will be used by default.
Specify indices of available GPUs by cuda environmental variable.

```
python train.py --device 2,3 -c config.json
```

This is equivalent to

```
CUDA_VISIBLE_DEVICES=2,3 python train.py -c config.py
```

## Customization

### Custom CLI options

Changing values of config file is a clean, safe and easy way of tuning hyperparameters. However, sometimes
it is better to have command line options if some values need to be changed too often or quickly.

This template uses the configurations stored in the json file by default, but by registering custom options as follows
you can change some of them using CLI flags.

```python
# simple class-like object having 3 attributes, `flags`, `type`, `target`.
CustomArgs = collections.namedtuple('CustomArgs', 'flags type target')
options = [
    CustomArgs(['--lr', '--learning_rate'], type=float, target=('optimizer', 'args', 'lr')),
    CustomArgs(['--bs', '--batch_size'], type=int, target=('data_loader', 'args', 'batch_size'))
    # options added here can be modified by command line flags.
]
```

`target` argument should be sequence of keys, which are used to access that option in the config dict. In this example, `target`
for the learning rate option is `('optimizer', 'args', 'lr')` because `config['optimizer']['args']['lr']` points to the learning rate.
`python train.py -c config.json --bs 256` runs training with options given in `config.json` except for the `batch size`
which is increased to 256 by command line options.

### Additional logging

If you have additional information to be logged, in `_train_epoch()` of your trainer class, merge them with `log` as shown below before returning:

```python
additional_log = {"gradient_norm": g, "sensitivity": s}
log.update(additional_log)
return log
```

### Testing

You can test trained model by running `eval.py` passing path to the trained checkpoint by `--resume` argument.

### Checkpoints

You can specify the name of the training session in config files:

```json
"name": "MNIST_LeNet",
```

The checkpoints will be saved in `save_dir/name/timestamp/checkpoint_epoch_n`, with timestamp in mmdd_HHMMSS format.

A copy of config file will be saved in the same folder.

**Note**: checkpoints contain:

```python
{
  'arch': arch,
  'epoch': epoch,
  'state_dict': self.model.state_dict(),
  'optimizer_G': self.optimizer_G.state_dict(),
  'optimizer_D': self.optimizer_D.state_dict(),
  'config': self.config
}
```

### Tensorboard Visualization

This template supports Tensorboard visualization by using either `torch.utils.tensorboard` or [TensorboardX](https://github.com/lanpa/tensorboardX).

1. **Install**

   If you are using pytorch 1.1 or higher, install tensorboard by 'pip install tensorboard>=1.14.0'.

   Otherwise, you should install tensorboardx. Follow installation guide in [TensorboardX](https://github.com/lanpa/tensorboardX).

2. **Run training**

   Make sure that `tensorboard` option in the config file is turned on.

   ```
    "tensorboard" : true
   ```

3. **Open Tensorboard server**

   Type `tensorboard --logdir saved/log/` at the project root, then server will open at `http://localhost:6006`

By default, values of loss and metrics specified in config file, input images, and histogram of model parameters will be logged.
If you need more visualizations, use `add_scalar('tag', data)`, `add_image('tag', image)`, etc in the `trainer._train_epoch` method.
`add_something()` methods in this template are basically wrappers for those of `tensorboardX.SummaryWriter` and `torch.utils.tensorboard.SummaryWriter` modules.

**Note**: You don't have to specify current steps, since `WriterTensorboard` class defined at `logger/visualization.py` will track current steps.

## Acknowledgements

This project is based on previous work by [victoresque](https://github.com/victoresque) on [PyTorch Template](https://github.com/victoresque/pytorch-template).
