## Customize Runtime Settings

### Customize optimization settings

#### Customize optimizer supported by Pytorch

We already support to use all the optimizers implemented by PyTorch, and the only modification is to change the `optimizer` field of config files.
For example, if you want to use `ADAM`, the modification could be as the following.

```python
optimizer = dict(type='Adam', lr=0.0003, weight_decay=0.0001)
```

To modify the learning rate of the model, the users only need to modify the `lr` in the config of optimizer. The users can directly set arguments following the [API doc](https://pytorch.org/docs/stable/optim.html?highlight=optim#module-torch.optim) of PyTorch.

#### Customize self-implemented optimizer

##### 1. Define a new optimizer

A customized optimizer could be defined as following.

Assume you want to add a optimizer named `MyOptimizer`, which has arguments `a`, `b`, and `c`.
You need to create a new file named `mmtrack/core/optimizer/my_optimizer.py`.

```python
from torch.optim import Optimizer
from mmcv.runner.optimizer import OPTIMIZERS


@OPTIMIZERS.register_module()
class MyOptimizer(Optimizer):

    def __init__(self, a, b, c)

```

##### 2. Add the optimizer to registry

To find the above module defined above, this module should be imported into the main namespace at first. There are two options to achieve it.

- Modify `mmtrack/core/optimizer/__init__.py` to import it.

    The newly defined module should be imported in `mmtrack/core/optimizer/__init__.py` so that the registry will find the new module and add it:

    ```python
    from .my_optimizer import MyOptimizer
    ```

- Use `custom_imports` in the config to manually import it

    ```python
    custom_imports = dict(imports=['mmtrack.core.optimizer.my_optimizer.py'], allow_failed_imports=False)
    ```

The module `mmtrack.core.optimizer.my_optimizer.MyOptimizer` will be imported at the beginning of the program and the class `MyOptimizer` is then automatically registered.
Note that only the package containing the class `MyOptimizer` should be imported.
`mmtrack.core.optimizer.my_optimizer.MyOptimizer` **cannot** be imported directly.

Actually users can use a totally different file directory structure using this importing method, as long as the module root can be located in `PYTHONPATH`.

##### 3. Specify the optimizer in the config file

Then you can use `MyOptimizer` in `optimizer` field of config files.
In the configs, the optimizers are defined by the field `optimizer` like the following:

```python
optimizer = dict(type='SGD', lr=0.02, momentum=0.9, weight_decay=0.0001)
```

To use your own optimizer, the field can be changed to

```python
optimizer = dict(type='MyOptimizer', a=a_value, b=b_value, c=c_value)
```

#### Customize optimizer constructor

Some models may have some parameter-specific settings for optimization, e.g. weight decay for BatchNorm layers.
The users can do those fine-grained parameter tuning through customizing optimizer constructor.

```python
from mmcv.utils import build_from_cfg

from mmcv.runner.optimizer import OPTIMIZER_BUILDERS, OPTIMIZERS
from mmtrack.utils import get_root_logger
from .my_optimizer import MyOptimizer


@OPTIMIZER_BUILDERS.register_module()
class MyOptimizerConstructor(object):

    def __init__(self, optimizer_cfg, paramwise_cfg=None):

    def __call__(self, model):

        return my_optimizer

```

The default optimizer constructor is implemented [here](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.DefaultOptimizerConstructor), which could also serve as a template for new optimizer constructor.

#### Additional settings

Tricks not implemented by the optimizer should be implemented through optimizer constructor (e.g., set parameter-wise learning rates) or hooks. We list some common settings that could stabilize the training or accelerate the training. Feel free to create PR, issue for more settings.

- __Use gradient clip to stabilize training__:
    Some models need gradient clip to clip the gradients to stabilize the training process. An example is as below:

    ```python
    optimizer_config = dict(
        _delete_=True, grad_clip=dict(max_norm=35, norm_type=2))
    ```

    If your config inherits the base config which already sets the `optimizer_config`, you might need `_delete_=True` to overide the unnecessary settings. See the [config documenetation](https://mmcv.readthedocs.io/en/latest/understand_mmcv/config.html#inherit-from-base-config-with-ignored-fields) for more details.

- __Use momentum schedule to accelerate model convergence__:
    We support momentum scheduler to modify model's momentum according to learning rate, which could make the model converge in a faster way.
    Momentum scheduler is usually used with LR scheduler, for example, the following config is used in 3D detection to accelerate convergence.
    For more details, please refer to the implementation of [CyclicLrUpdater](https://github.com/open-mmlab/mmcv/blob/f48241a65aebfe07db122e9db320c31b685dc674/mmcv/runner/hooks/lr_updater.py#L327) and [CyclicMomentumUpdater](https://github.com/open-mmlab/mmcv/blob/f48241a65aebfe07db122e9db320c31b685dc674/mmcv/runner/hooks/momentum_updater.py#L130).

    ```python
    lr_config = dict(
        policy='cyclic',
        target_ratio=(10, 1e-4),
        cyclic_times=1,
        step_ratio_up=0.4,
    )
    momentum_config = dict(
        policy='cyclic',
        target_ratio=(0.85 / 0.95, 1),
        cyclic_times=1,
        step_ratio_up=0.4,
    )
    ```

### Customize training schedules

We support many other learning rate schedule [here](https://github.com/open-mmlab/mmcv/blob/master/mmcv/runner/hooks/lr_updater.py), such as `CosineAnnealing` and `Poly` schedule. Here are some examples

- Poly schedule:

    ```python
    lr_config = dict(policy='poly', power=0.9, min_lr=1e-4, by_epoch=False)
    ```

- ConsineAnnealing schedule:

    ```python
    lr_config = dict(
        policy='CosineAnnealing',
        warmup='linear',
        warmup_iters=1000,
        warmup_ratio=1.0 / 10,
        min_lr_ratio=1e-5)
    ```

### Customize workflow

Workflow is a list of (phase, epochs) to specify the running order and epochs.
By default it is set to be

```python
workflow = [('train', 1)]
```

which means running 1 epoch for training.
Sometimes user may want to check some metrics (e.g. loss, accuracy) about the model on the validate set.
In such case, we can set the workflow as

```python
[('train', 1), ('val', 1)]
```

so that 1 epoch for training and 1 epoch for validation will be run iteratively.

**Note**:

1. The parameters of model will not be updated during val epoch.
2. Keyword `total_epochs` in the config only controls the number of training epochs and will not affect the validation workflow.
3. Workflows `[('train', 1), ('val', 1)]` and `[('train', 1)]` will not change the behavior of `EvalHook` because `EvalHook` is called by `after_train_epoch` and validation workflow only affect hooks that are called through `after_val_epoch`. Therefore, the only difference between `[('train', 1), ('val', 1)]` and `[('train', 1)]` is that the runner will calculate losses on validation set after each training epoch.

### Customize hooks

#### Customize self-implemented hooks

##### 1. Implement a new hook

There are some occasions when the users might need to implement a new hook. MMTracking supports customized hooks in training. Thus the users could implement a hook directly in mmtrack or their mmtrack-based codebases and use the hook by only modifying the config in training.
Here we give an example of creating a new hook in mmtrack and using it in training.

```python
from mmcv.runner import HOOKS, Hook


@HOOKS.register_module()
class MyHook(Hook):

    def __init__(self, a, b):
        pass

    def before_run(self, runner):
        pass

    def after_run(self, runner):
        pass

    def before_epoch(self, runner):
        pass

    def after_epoch(self, runner):
        pass

    def before_iter(self, runner):
        pass

    def after_iter(self, runner):
        pass
```

Depending on the functionality of the hook, the users need to specify what the hook will do at each stage of the training in `before_run`, `after_run`, `before_epoch`, `after_epoch`, `before_iter`, and `after_iter`.

##### 2. Register the new hook

Then we need to make `MyHook` imported. Assuming the file is in `mmtrack/core/utils/my_hook.py` there are two ways to do that:

- Modify `mmtrack/core/utils/__init__.py` to import it.

    The newly defined module should be imported in `mmtrack/core/utils/__init__.py` so that the registry will find the new module and add it:

    ```python
    from .my_hook import MyHook
    ```

- Use `custom_imports` in the config to manually import it

    ```python
    custom_imports = dict(imports=['mmtrack.core.utils.my_hook'], allow_failed_imports=False)
    ```

##### 3. Modify the config

```python
custom_hooks = [
    dict(type='MyHook', a=a_value, b=b_value)
]
```

You can also set the priority of the hook by adding key `priority` to `'NORMAL'` or `'HIGHEST'` as below

```python
custom_hooks = [
    dict(type='MyHook', a=a_value, b=b_value, priority='NORMAL')
]
```

By default the hook's priority is set as `NORMAL` during registration.

#### Use hooks implemented in MMCV

If the hook is already implemented in MMCV, you can directly modify the config to use the hook as below

```python
custom_hooks = [
    dict(type='MyHook', a=a_value, b=b_value, priority='NORMAL')
]
```

#### Modify default runtime hooks

There are some common hooks that are not registerd through `custom_hooks`, they are

- log_config
- checkpoint_config
- evaluation
- lr_config
- optimizer_config
- momentum_config

In those hooks, only the logger hook has the `VERY_LOW` priority, others' priority are `NORMAL`.
The above-mentioned tutorials already covers how to modify `optimizer_config`, `momentum_config`, and `lr_config`.
Here we reveals how what we can do with `log_config`, `checkpoint_config`, and `evaluation`.

##### Checkpoint hook

The MMCV runner will use `checkpoint_config` to initialize [`CheckpointHook`](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.CheckpointHook).

```python
checkpoint_config = dict(interval=1)
```

The users could set `max_keep_ckpts` to only save only small number of checkpoints or decide whether to store state dict of optimizer by `save_optimizer`. More details of the arguments are [here](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.CheckpointHook)

##### Log hook

The `log_config` wraps multiple logger hooks and enables to set intervals. Now MMCV supports `WandbLoggerHook`, `MlflowLoggerHook`, and `TensorboardLoggerHook`.
The detail usages can be found in the [doc](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.LoggerHook).

```python
log_config = dict(
    interval=50,
    hooks=[
        dict(type='TextLoggerHook'),
        dict(type='TensorboardLoggerHook')
    ])
```

##### Evaluation hook

The config of `evaluation` will be used to initialize the [`EvalHook`](https://mmcv.readthedocs.io/en/latest/api.html#mmcv.runner.EvalHook).
Except keys like `interval`, `start` and so on, other arguments such as `metric` will be passed to the `dataset.evaluate()`

```python
evaluation = dict(interval=1, metric='bbox')
```
