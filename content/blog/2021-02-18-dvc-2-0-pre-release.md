---
title: DVC 2.0 Pre-Release
date: 2021-02-17
description: |
  Today, we're announcing DVC 2.0 pre-release. We'll share lessons from our
  journey and how these will be reflected in the coming release.

descriptionLong: |
  The new release is a result of our learning from our users. There are four
  major features are coming:

  🔗 ML pipeline templating and iterative foreach stages

  🧪 Lightweight ML experiments

  📍 ML model checkpoints

  📈 Dvc-live - new open-source library for metrics logging

picture: 2021-02-18/dvc-2-0-pre-release.png
pictureComment: DVC 2.0 Pre-Release
author: dmitry_petrov
commentsUrl: https://discuss.dvc.org/t/dvc-3-years-anniversary-and-1-0-pre-release/374
tags:
  - Release
  - MLOps
  - DataOps
---

## Install

First things first. You can install the 2.0 pre-release from the master branch
in our repo (instruction [here](https://dvc.org/doc/install/pre-release)) or
through pip:

```dvc
$ pip install --upgrade --pre dvc
```

## ML pipelines parameterization and foreach stages

After introducing the multi-stage pipeline file `dvc.yaml`, it was quickly
adopted among our users. The DVC team got tons of positive feedback from them,
as well as feature requests.

### Pipeline parameters from `vars`

The most requested feature was the ability to use parameters in `dvc.yaml`. For
example. So, you can pass the same seed value or filename to multiple stages in
the pipeline.

```yaml
vars:
    train_matrix: train.pkl
    test_matrix: test.pkl
    seed: 20210215

...

stages:
    process:
        cmd: python process.py \
                --seed ${seed} \
                --train ${train_matrix} \
                --test ${test_matrix}
        outs:
        - ${test_matrix}
        - ${train_matrix}

        ...

    train:
        cmd: python train.py ${train_matrix} --seed ${seed}
        deps:
        - ${train_matrix}
```

Also, it gives an ability to localize all important parameters in a single
`vars` block, and play with them. This is a natural thing to do for scenarios
like NLP or when hyperparameter optimization is happening not only in the model
training code but in the data processing as well.

### Pipeline parameters from params files

It is quite common to define pipeline parameters in a config file or a
parameters file (like `params.yaml`) instead of in the pipeline file `dvc.yaml`
itself. These parameters defined in `params.yaml` can also be used in
`dvc.yaml`.

```yaml
# params.yaml
models:
  us:
    thresh: 10
    filename: 'model-us.hdf5'
```

```yaml
# dvc.yaml
stages:
  build-us:
    cmd: >-
      python script.py
        --out ${models.us.filename}
        --thresh ${models.us.thresh}
    outs:
      - ${models.us.filename}
```

DVC properly tracks params dependencies for each stage starting from the
previous DVC version 1.0. See the
[`--params` option](/doc/command-reference/run#for-displaying-and-comparing-data-science-experiments)
of `dvc run` for more details.

### Iterating over params with foreach stages

Iterating over params was a frequently requested feature. Now users can define
multiple similar stages with a templatized command.

```yaml
stages:
  build:
    foreach:
      gb:
        thresh: 15
        filename: 'model-gb.hdf5'
      us:
        thresh: 10
        filename: 'model-us.hdf5'
    do:
      cmd: >-
        python script.py --out ${item.filename} --thresh ${item.thresh}
      outs:
        - ${item.filename}
```

## Lightweight ML experiments

DVC uses Git as a foundation for ML experiments. This solid foundation makes
each ML experiment reproducible and accessible from Git history. This Git-based
approach works very well for ML projects with mature ML models when only a few
new experiments per day are running. However, in more active development when
dozens or hundreds of experiments need to be run in a single day, Git creates
overhead - each experiment run requires additional Git commands
`git add/commit`, and comparing all experiments is difficult.

We introduce lightweight experiments in DVC 2.0! This is the way of
auto-tracking without any overhead from ML engineers.

⚠️ Note, ML experiment is an experimental feature in the coming release. It
means the commands might change a bit even after the release.

Run an ML experiment with a new hyperparameter from `params.yaml`:

```dvc
$ dvc exp run --set-param featurize.max_features=3000

Reproduced experiment(s): exp-bb55c
Experiment results have been applied to your workspace.

$ dvc exp diff
Path         Metric    Value    Change
scores.json  auc       0.57462  0.0072197

Path         Param                   Value    Change
params.yaml  featurize.max_features  3000     1500
```

More experiments:

```dvc
$ dvc exp run --set-param featurize.max_features=4000
Reproduced experiment(s): exp-9bf22
Experiment results have been applied to your workspace.

$ dvc exp run --set-param featurize.max_features=5000
Reproduced experiment(s): exp-63ee0
Experiment results have been applied to your workspace.

$ dvc exp run --set-param featurize.max_features=5000 \
                --set-param featurize.ngrams=3
Reproduced experiment(s): exp-80655
Experiment results have been applied to your workspace.
```

In the examples above, hyperparamteres were changed automaticaly by option
`--set-param`. User can make this changes manualy by modifying the file. The
same way _any code or data files can be changed_ and `dvc exp run` will capture
the changes.

See all the runs:

```dvc
$ dvc exp show --no-pager --no-timestamp \
        --include-params featurize.max_features,featurize.ngrams
┏━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┓
┃ Experiment    ┃     auc ┃ featurize.max_features ┃ featurize.ngrams ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━┩
│ workspace     │ 0.56359 │ 5000                   │ 3                │
│ master        │  0.5674 │ 1500                   │ 2                │
│ ├── exp-80655 │ 0.56359 │ 5000                   │ 3                │
│ ├── exp-63ee0 │  0.5515 │ 5000                   │ 2                │
│ ├── exp-9bf22 │ 0.56448 │ 4000                   │ 2                │
│ └── exp-bb55c │ 0.57462 │ 3000                   │ 2                │
└───────────────┴─────────┴────────────────────────┴──────────────────┘
```

No artificial branches, only custome references `exps` (do not worry if you
don't understand this part - it is an implementation detail):

```dvc
$ git branch
* master

$ git show-ref
5649f62d845fdc29e28ea6f7672dd729d3946940 refs/exps/exec/EXEC_APPLY
5649f62d845fdc29e28ea6f7672dd729d3946940 refs/exps/exec/EXEC_BRANCH
5649f62d845fdc29e28ea6f7672dd729d3946940 refs/exps/71/67904d89e116f28daf7a6e4c0878268117c893/exp-80655
f16e7b7c804cf52d91d1d11850c15963fb2a8d7b refs/exps/97/d69af70c6fb4bc59aefb9a87437dcd28b3bde4/exp-63ee0
0566d42cddb3a8c4eb533f31027f0febccbbc2dd refs/exps/91/94265d5acd847e1c439dd859aa74b1fc3d73ad/exp-bb55c
9bb067559583990a8c5d499d7435c35a7c9417b7 refs/exps/49/5c835cd36772123e82e812d96eabcce320f7ec/exp-9bf22
```

The best experiment can be promoted to the workspace and commited to Git.

```dvc
$ dvc exp apply exp-bb55c
$ git add .
$ git commit -m 'optimize max feature size'
```

Alternatively, an experiment can be promoted to a branch (`big_fr_size` branch
in this case):

```dvc
$ dvc exp branch exp-80655 big_fr_size
Git branch 'big_fr_size' has been created from experiment 'exp-c695f'.
To switch to the new branch run:

	git checkout big_fr_size
```

Remove all the experiments that were not used:

```dvc
$ dvc exp gc --workspace --force
```

## Model checkpoints tracking

ML model checkpoints are an essential part of deep learning. ML engineers prefer
to save the model files (or weights) at checkpoints during a training process
and return back when metrics start diverging or learning is not fast enough.

The checkpoints create a different dynamic around ML modeling process and need a
special support from the toolset:

1. Track and save model checkpoints (DVC outputs) periodically, not only the
   final result or training epoch.
2. Save metrics corresponding to each of the checkpoints.
3. Reuse checkpoints - warm-start training with an existing model file,
   corresponding code, dataset version and metrics.

This new behaviour is supported in DVC 2.0. Now, DVC can version all your
checkpoints with corresponding code and data. It brings reproducibility of DL
processes to the next level - every checkpoint is reproducible.

This is how you define checkpoints with live-metrics:

```

```

Start and interrupt training process:

```

```

Navigate in checkpoints:

```

```

Get back to any version:

```

```

Under the hood DVC uses Git to store the checkpoint meta-information.
Straight-forward implementation of checkpoints on top of Git should include
branches and auto-commits in the branches. This approach over-pollutes the
branch namespace very quickly. To avoid this issue, we introduced Git custom
references `exps` the same way as GitHub uses Git custom references `pulls` to
track pull requests. This is an interesting technical topic that deserves a
separate blog post. Please follow us if you are interested.

## Metrics logging

Continuously logging ML metrics is a very common practice in the ML world.
Instead of a simple command line output with the metrics values many ML
engineers prefer visuals and plots. These plots can be organized in a "database"
of ML experiments to keep track of a project. There are many special solutions
for metrics collecting and experiment tracking such as sacred, mlflow, weight
and biases, neptune.ai or other.

With DVC 2.0 we are releasing new open-source library
[DVC-Live](https://github.com/iterative/dvclive) that provide functionality for
tracking model metrics and organizing metrics in simple text files in a way that
DVC can visualize the metrics with navigation in Git histroy. So, DVC can show
you a metrics difference between current model and a model in `master` or any
other branch.

This approach is similar to the other metrics tracking tools with the difference
that Git becomes a "database" or of ML experiments.

### Generate metrics file

Install the library:

```dvc
$ pip install dvclive
```

Instrument your code:

```python
import dvclive
from dvclive.keras import DvcLiveCallback

dvclive.init("logs") #, summarize=True)

...

model.fit(...
          # Set up DVC-Live callback:
          callbacks=[ DvcLiveCallback() ]
         )

```

During the training you will see the metrics files that are continiously
populated each epoches:

```dvc
$ ls logs/
accuracy.tsv     loss.tsv         val_accuracy.tsv val_loss.tsv

$ head logs/accuracy.tsv
timestamp	step	accuracy
1613645582716	0	0.7360000014305115
1613645585478	1	0.8349999785423279
1613645587322	2	0.8830000162124634
1613645589125	3	0.9049999713897705
1613645590891	4	0.9070000052452087
1613645592681	5	0.9279999732971191
1613645594490	6	0.9430000185966492
1613645596232	7	0.9369999766349792
1613645598034	8	0.9430000185966492
```

In addition to the continious metrics files you will see the summary metrics
file and html file with the same file prefix. The summary file conteins the
result of the latest epoch:

```dvc
$ cat logs.json | python -m json.tool
{
    "step": 41,
    "loss": 0.015958430245518684,
    "accuracy": 0.9950000047683716,
    "val_loss": 13.705962181091309,
    "val_accuracy": 0.5149999856948853
}
```

The html file contains all the visuals for continious metrics as well as the
summary metrics in a single page:

![](/uploads/images/2021-02-18/dvclive-html.png)

Note, the HTML and the summary metrics files are generating automatically for
each. So, you can monitor model performance in realtime.

### Navigation with the metrics file

DVC repository is NOT required to use the live metrics functionality from the
above. It works independently from DVC.

DVC repository become usefule when the metrics and plots are commited in your
Git repository and you need navigation around the metrics.

```dvc
$ dvc metrics diff
????
```

The same with the plots:

```dvc
$ dvc plots diff ....
????
```

Another nice thing about the live metrics - they work across ML experiments and
checkpoints if properly set up in dvc stages. To set up live metrics you need to
specify the metrics directory in `live` section of a stage:

```yaml
stages:
  train:
    cmd: python train.py
    live:
      logs:
        cache: false
        summary: true
        report: true
    deps:
      - data
```

## Thank you!

I'd like to thank all of you DVC community members for the feedback that we are
constantly getting. This feedback helps us build new functionalities in DVC and
make it more stable.

Please be in touch with us on [Twitter](https://twitter.com/DVCorg) and our
[Discord channel](https://dvc.org/chat).
