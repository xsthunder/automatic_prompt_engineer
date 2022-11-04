# Large Language Models Are Human-Level Prompt Engineers

Yongchao Zhou*, Andrei Ioan Muresanu*, Ziwen Han\*, Keiran Paster, Silviu Pitis, Harris Chan, Jimmy Ba

[Project Page](https://sites.google.com/view/automatic-prompt-engineer) | [ArXiv](https://arxiv.org/abs/2211.01910)

This repo contains code for "Large Language Models Are Human-Level Prompt Engineers". Please see our
paper and project page for more results.

# Abstract

By conditioning on natural language instructions, large language models (LLMs) have displayed impressive capabilities as
general-purpose computers. However, task performance depends significantly on the quality of the prompt used to steer
the model, and most effective prompts have been handcrafted by humans. Inspired by classical program synthesis and the
human approach to prompt engineering, we propose Automatic Prompt Engineer (APE) for automatic instruction generation
and selection. In our method, we treat the instruction as the “program,” optimized by searching over a pool of
instruction candidates proposed by an LLM in order to maximize a chosen score function. To evaluate the quality of the
selected instruction, we evaluate the zero-shot performance of another LLM following the selected instruction.
Experiments on 24 NLP tasks show that our automatically generated instructions outperform the prior LLM baseline by a
large margin and achieve better or comparable performance to the instructions generated by human annotators on 21/24
tasks. We conduct extensive qualitative and quantitative analyses to explore the performance of APE. We show that
APE-engineered prompts can be applied to steer models toward truthfulness and/or informativeness, as well as to improve
few-shot learning performance by simply prepending them to standard in-context learning prompts.

## Installation

To install APE, simply run:

```
pip install -e .
```

And add your OPENAI_API_KEY with the following command:

```
export OPENAI_API_KEY=YOUR_KEY
```

## Using `APE`

APE comes with two interfaces, the `find_prompts` function and a simplified version which takes care of most of the configuration for you.

### Templates :memo:

APE is built around three types of templates: evaluation templates, prompt generation templates, and demonstration templates.

The evaluation template (`eval_template`) defines the format of the input to the language model used to evaluate the quality of different prompts. The template supports the following tokens:

- [PROMPT] - The prompt to evaluate
- [INPUT] - The input to the model
- [OUTPUT] - The target output of the model
- [full_DEMO] - Demonstrations (if you want to use few-shot learning)

For example, the zero-shot evaluation template for the instruction induction tasks is:

```
Instruction: [PROMPT]
Input: [INPUT]
Output: [OUTPUT]
```

For instruction + few-shot learning, the template is:

```
Instruction: [PROMPT]

[full_DEMO]

Input: [INPUT]
Output: [OUTPUT]
```

The prompt generation template (`prompt_gen_template`) defines the format of the input to the language model used to generate candidate prompts. The template supports the following tokens:

- [APE] - A token to be replaced by the LLM's generated text
- [full_DEMO] - Demonstrations of input/output pairs
- [INPUT] - The input of the first demonstration
- [OUTPUT] - The output of the first demonstration

For example, in the instruction induction task, we generate candidate prompts using the following template:

```
I gave a friend an instruction. Based on the instruction they produced the following input-output pairs:

[full_DEMO]

The instruction was to [APE]
```

:warning: By default, `prompt_gen_template` is set to `None`. In this case, if the `prompt_gen_mode` is `forward`, the above tepmlate is used. If the `prompt_gen_mode` is `insert`, the template is simply converted from the evaluation template by replacing `[PROMPT]` with `[APE]`.

Finally, the demonstration template (`demos_template`) defines the format of the demonstrations. The template supports the following tokens:

- [INPUT] - The input of the demonstration
- [OUTPUT] - The output of the demonstration

### Data :card_index:

Datasets in this codebase are represented using separate lists for inputs and outputs. For example, a dataset for words and their antonyms can be written as:

```python
words = ["sane", "direct", "informally", "unpopular", "subtractive", "nonresidential",
    "inexact", "uptown", "incomparable", "powerful", "gaseous", "evenly", "formality",
    "deliberately", "off"]
antonyms = ["insane", "indirect", "formally", "popular", "additive", "residential",
    "exact", "downtown", "comparable", "powerless", "solid", "unevenly", "informality",
    "accidentally", "on"]
data = (words, antonyms)
```

### `find_prompts`

The `find_prompts` function takes the following arguments:

- `eval_template` - The evaluation template
- `demos_template` - The demonstration template
- `prompt_gen_data` - The data used to generate candidate prompts
- `eval_data` - The data used to evaluate the quality of candidate prompts
- `conf` - A dictionary of configuration options
- `base_conf` - The path to the base configuration dataset.
  - `'configs/default.yaml'`: Default configuration that uses likelihood to evaluate prompts.
  - `'configs/bandits.yaml'`: Configuration that uses the Upper Confidence Bound algorithm to save resources when evaluating prompts. Uses likelihood as its base evaluation method.
- `few_shot_data` - The data used for few-shot learning
- `prompt_gen_template` - The prompt generation template

For more information on the `conf` dictionary, please refer to the annotations in `configs/bandits.yaml`.

`find_prompts` returns an evaluation result object and a demo function to allow you to evaluate the quality of the selected prompt manually. To get the best prompts and their associated scores from the evaluation result object, use the `sorted()` method.

### `simple_ape`

The `simple_ape` function takes the following arguments:

- `dataset` - The dataset to use (simple APE uses the same dataset for prompt generation, evaluation, and few-shot learning)
- `eval_template` - The evaluation template (default: `'Instruction: [PROMPT]\nInput: [INPUT]\nOutput: [OUTPUT]'`)
- `prompt_gen_template` - The prompt generation template (default: `None`)
- `prompt_gen_mode` - The mode to use for prompt generation (default: `'forward'`)
- `demos_template` - The demonstration template (default: `'Input: [INPUT]\nOutput: [OUTPUT]'`)
- `eval_model` - The model to use for evaluation (default: `'text-davinci-002'`)
- `prompt_gen_model` - The model to use for prompt generation (default: `'text-davinci-002'`)
- `num_prompts` - The number of candidate prompts to generate (default: `50`)
- `eval_rounds` - The number of UCB rounds of evaluation to perform (default: `20`)
- `prompt_gen_batch_size` - The batch size to use for prompt generation (default: `200`)
- `eval_batch_size` - The batch size to use for evaluation (default: `500`)

`simple_ape` is designed to simplify many of the choices that need to be made when using `find_prompts`. Particularly, it simplifies evaluation by choosing only the amount of candidate prompts to generate and the number of rounds of UCB to run. Behind the scenes, `simple_ape` uses `num_prompts_per_round` equal to `num_prompts // 3` and fixes the number of samples per prompt per round to `5`.

An example usage of this function would look like:

```python
from automatic_prompt_engineer import ape

words = ["sane", "direct", "informally", "unpopular", "subtractive", "nonresidential",
    "inexact", "uptown", "incomparable", "powerful", "gaseous", "evenly", "formality",
    "deliberately", "off"]
antonyms = ["insane", "indirect", "formally", "popular", "additive", "residential",
    "exact", "downtown", "comparable", "powerless", "solid", "unevenly", "informality",
    "accidentally", "on"]

result, demo_fn = ape.simple_ape(
    dataset=(words, antonyms),
    eval_template=eval_template,
)
```

`find_prompts` returns an evaluation result object and a demo function to allow you to evaluate the quality of the selected prompt manually. To get the best prompts and their associated scores from the evaluation result object, use the `sorted()` method.

### Cost Estimation

As APE can often be expensive to run, we provide cost estimations for `find_prompts` and `simple_ape`. Simply use `ape.estimate_cost` or `ape.simple_estimate_cost` with the same arguments as `find_prompts` and `simple_ape` respectively.

## Try it out! :eyes:

We provide a colab notebook for easily using APE:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1Hrz6Q7GFdH5OVg3Dis86f5OqiGdkDfRP?usp=sharing)

We also provide a GUI for easily using APE. Please follow the instructions in the following colab to run it:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1oL1CcvzRybAbmeqs--2csaIvSOpjH072?usp=sharing)

## Code Strucutre

```
- automatic_prompt_engineer
    |- configs
    |- evaluation
    |- ape.py
    |- config.py
    |- evaluate.py
    |- generate.py
    |- llm.py
    |- template.py
- experiments: scripts for experiments
    |- configs
    |- data
    |- evaluation
    |- run_instruction_induction.py
    |- run_truthful_qa.py
- tests: unit tests
- demo.py: script for launching the GUI
- demo.ipynb: notebook demonstrating simple_ape
```

## Reproducing Experiments :test_tube:

To reproduce the experiments from the paper, simply run the scripts in the `experiments` folder. For example, to reproduce the experiments for the instruction induction task, run:

`python experiments/run_instruction_induction.py --task=antonyms`

To run the TruthfulQA experiment, run:

`python experiments/run_truthful_qa.py`

## Comments

Our codebase is based on the following repo. Thanks for open-sourcing!

- [Instruction Induction](https://github.com/orhonovich/instruction-induction).
- [TruthfulQA](https://github.com/sylinrl/TruthfulQA)

## BibTeX

```
@article{zhou2022large,
      title={Large Language Models Are Human-Level Prompt Engineers}, 
      author={Yongchao Zhou and Andrei Ioan Muresanu and Ziwen Han and Keiran Paster and Silviu Pitis and Harris Chan and Jimmy Ba},
      year={2022},
      eprint={2211.01910},
      archivePrefix={arXiv},
      primaryClass={cs.LG}
}
```
