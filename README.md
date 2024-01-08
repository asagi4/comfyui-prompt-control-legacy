# ComfyUI prompt control

Nodes for convenient prompt editing. The aim is to make basic generations in ComfyUI completely prompt-controllable.

The basic nodes should now be stable, though I won't make interface guarantees quite yet.

## What can it do?

Things you can control via the prompt:
- Prompt editing and filtering without multiple samplers
- LoRA loading and scheduling (including LoRA block weights)
- Prompt masking and area control, combining prompts and interpolation
- SDXL parameters
- Other miscellaneous things

[This example workflow](workflows/example.json?raw=1) implements a two-pass workflow illustrating most scheduling features.

## Requirements

You need to have `lark` installed in your Python environment for parsing to work (If you reuse A1111's venv, it'll already be there)

If you use the portable version of ComfyUI on Windows with its embedded Python, you must open a terminal in the ComfyUI installation directory and run the command:
```
.\python_embeded\python.exe -m pip install lark
```

Then restart ComfyUI afterwards.

## Notable changes

I try to avoid behavioural changes that break old prompts, but they may happen occasionally.
- 2024-01-08 I don't recommend or test AITemplate anymore. Use Stable-Fast instead
- 2023-12-28 MASK now uses ComfyUI's `mask_strength` attribute instead of calculating it on its own. This changes its behaviour slightly.
- 2023-12-06: Removed `JinjaRender`, `SimpleWildcard`, `ConditioningCutoff`, `CondLinearInterpolate` and `StringConcat`. For the first two, see [this repository](https://github.com/asagi4/comfyui-utility-nodes) for mostly-compatible implementations.
- 2023-10-04: `STYLE:...` syntax changed to `STYLE(...)`

## Note on how schedules work

ComfyUI does not use the step number to determine whether to apply conds; instead, it uses the sampler's timestep value which is affected by the scheduler you're using. This means that when the sampler scheduler isn't linear, the schedules generated by prompt control will not be either.

Currently there doesn't seem to be a good way to change this.

You can try using the `PCSplitSampling` node to enable an alternative method of sampling.

# Scheduling syntax

Syntax is like A1111 for now, but only fractions are supported for steps.

```
a [large::0.1] [cat|dog:0.05] [<lora:somelora:0.5:0.6>::0.5]
[in a park:in space:0.4]
```

You can also use `a [b:c:0.3,0.7]` as a shortcut. The prompt be `a` until 0.3, `a b` until 0.7, and then `c`. `[a:0.1,0.4]` is equivalent to `[a::0.1,0.4]`

## Alternating

Alternating syntax is `[a|b:pct_steps]`, causing the prompt to alternate every `pct_steps`. `pct_steps` defaults to 0.1 if not specified. You can also have more than two options.

## Sequences

The syntax `[SEQ:a:N1:b:N2:c:N3]` is shorthand for `[a:[b:[c::N3]:N2]:N1]` ie. it switches from `a` to `b` to `c` to nothing at the specified points in sequence.

Might be useful with Jinja templating (see https://github.com/asagi4/comfyui-utility-nodes). For example:
```
[SEQ<% for x in steps(0.1, 0.9, 0.1) %>:<lora:test:<= sin(x*pi) + 0.1 =>>:<= x =><% endfor %>]
```
generates a LoRA schedule based on a sinewave

## Tag selection
Instead of step percentages, you can use a *tag* to select part of an input:
```
a large [dog:cat<lora:catlora:0.5>:SECOND_PASS]
```
You can then use the `tags` parameter in the `FilterSchedule` node to filter the prompt. If the tag matches any tag `tags` (comma-separated), the second option is returned (`cat`, in this case, with the LoRA). Otherwise, the first option is chosen (`dog`, without LoRA).

the values in `tags` are case-insensitive, but the tags in the input **must** be uppercase A-Z and underscores only, or they won't be recognized. That is, `[dog:cat:hr]` will not work.

For example, a prompt
```
a [black:blue:X] [cat:dog:Y] [walking:running:Z] in space
```
with `tags` `x,z` would result in the prompt `a blue cat running in space`

## Prompt interpolation

`a red [INT:dog:cat:0.2,0.8:0.05]` will attempt to interpolate the tensors for `a red dog` and `a red cat` between the specified range in as many steps of 0.05 as will fit.


## SDXL

You can use the function `SDXL(width height, target_width target_height, crop_w crop_h)` to set SDXL prompt parameters. `SDXL()` is equivalent to `SDXL(1024 1024, 1024 1024, 0 0)`

To set the `clip_l` prompt, as with `CLIPTextEncodeSDXL`, use the function `CLIP_L(prompt text goes here)`. multiple instances of `CLIP_L` are concatenated, and `BREAK` isn't supported in it. It has no effect on SD 1.5. The rest of the prompt becomes the `clip_g` prompt.

if there is no `CLIP_L`, the prompts will work as with `CLIPTextEncode`.

# Other syntax:

- `<emb:xyz>` is alternative syntax for `embedding:xyz` to work around a syntax conflict with `[embedding:xyz:0.5]` which is parsed as a schedule that switches from `embedding` to `xyz`.

- The keyword `BREAK` causes the prompt to be tokenized in separate chunks, which results in each chunk being individually padded to the text encoder's maximum token length. This is mostly equivalent to the `ConditioningConcat` node.

## Combining prompts
`AND` can be used to combine prompts. You can also use a weight at the end. It does a weighted sum of each prompt,
```
cat :1 AND dog :2
```
The weight defaults to 1 and are normalized so that `a:2 AND b:2` is equal to `a AND b`. `AND` is processed after schedule parsing, so you can change the weight mid-prompt: `cat:[1:2:0.5] AND dog`

if there is `COMFYAND()` in the prompt, the behaviour of `AND` will change to work like `ConditioningCombine`, but in practice this seems to be just slower while producing the same output.

## Functions

There are some "functions" that can be included in a prompt to do various things. 

Like `AND`, these functions are parsed after regular scheduling syntax has been expanded, allowing things like `[AREA:MASK:0.3](...)`, in case that's somehow useful.

### NOISE

The function `NOISE(weight, seed)` adds some random noise into the prompt. The seed is optional, and if not specified, the global RNG is used. `weight` should be between 0 and 1.

### MASK and AREA
You can use `MASK(x1 x2, y1 y2, weight)` to specify a region mask for a prompt. The values are specified as a percentage with a float between `0` and `1`, or as absolute pixel values (these can't be mixed). `1` will be interpreted as a percentage instead of a pixel value.

Masks assume a size of `(512, 512)`, and pixel values will be relative to that. ComfyUI will scale the mask to match the image resolution, but you can change it manually by using `MASK_SIZE(width, height)` anywhere in the prompt,

These are handled per `AND`-ed prompt, so in `prompt1 AND MASK(...) prompt2`, the mask will only affect prompt2.

The default values are `MASK(0 1, 0 1, 1)` and you can omit unnecessary ones, that is, `MASK(0 0.5, 0.3)` is `MASK(0 0.5, 0.3 1, 1)`

Note that because the default values are percentages, `MASK(0 256, 64 512)` is valid, but `MASK(0 200)` will raise an error.

Similarly, you can use `AREA(x1 x2, y1 y2, weight)` to specify an area for the prompt (see ComfyUI's area composition examples). The area is calculated by ComfyUI relative to your latent size.

Masking does not affect LoRA scheduling unless you set unet weights to 0 for a LoRA.

# Schedulable LoRAs
The `ScheduleToModel` node patches a model such that when sampling, it'll switch LoRAs between steps. You can apply the LoRA's effect separately to CLIP conditioning and the unet (model).

For me this seems to be quite slow without the --highvram switch because ComfyUI will shuffle things between the CPU and GPU. YMMV. When things stay on the GPU, it's quite fast.

## LoRA Block Weight

If you have [ComfyUI Inspire Pack](https://github.com/ltdrdata/ComfyUI-Inspire-Pack) installed, you can use its Lora Block Weight syntax, for example:

```
a prompt <lora:cars:1:LBW=SD-OUTALL;A=1.0;B=0.0;>
```
The `;` is optional if there is only 1 parameter.
The syntax is the same as in the `ImpactWildcard` node, documented [here](https://github.com/ltdrdata/ComfyUI-extension-tutorials/blob/Main/ComfyUI-Impact-Pack/tutorial/ImpactWildcard.md)

# Other integrations
## Advanced CLIP encoding
You can use the syntax `STYLE(weight_interpretation, normalization)` in a prompt to affect how prompts are interpreted.

Without any extra nodes, only `perp` is available, which does the same as [ComfyUI_PerpWeight](https://github.com/bvhari/ComfyUI_PerpWeight) extension.

If you have [Advanced CLIP Encoding nodes](https://github.com/BlenderNeko/ComfyUI_ADV_CLIP_emb/tree/master) cloned into your `custom_nodes`, more options will be available.

The style can be specified separately for each AND:ed prompt, but the first prompt is special; later prompts will "inherit" it as default. For example:

```
STYLE(A1111) a (red:1.1) cat with (brown:0.9) spots and a long tail AND an (old:0.5) dog AND a (green:1.4) (balloon:1.1)
```
will interpret everything as A1111, but
```
a (red:1.1) cat with (brown:0.9) spots and a long tail AND STYLE(A1111) an (old:0.5) dog AND a (green:1.4) (balloon:1.1)
```
Will interpret the first one using the default ComfyUI behaviour, the second prompt with A1111 and the last prompt with the default again

For things (ie. the code imports) to work, the nodes must be cloned in a directory named exactly `ComfyUI_ADV_CLIP_emb`.

## Cutoff node integration

If you have [ComfyUI Cutoff](https://github.com/BlenderNeko/ComfyUI_Cutoff) cloned into your `custom_nodes`, you can use the `CUT` keyword to use cutoff functionality

The syntax is
```
a group of animals, [CUT:white cat:white], [CUT:brown dog:brown:0.5:1.0:1.0:_]
```
the parameters in the `CUT` section are `region_text:target_text:weight;strict_mask:start_from_masked:padding_token` of which only the first two are required.
If `strict_mask`, `start_from_masked` or `padding_token` are specified in more than one section, the last one takes effect for the whole prompt

## Stable-Fast

The prompt control node works well with [ComfyUI_stable_fast](https://github.com/gameltb/ComfyUI_stable_fast). However, you should apply `ScheduleToModel` **after** applying `Apply StableFast Unet` to prevent constant recompilations.

# Nodes

## PromptToSchedule
Parses a schedule from a text prompt. A schedule is essentially an array of `(valid_until, prompt)` pairs that the other nodes can use.

## FilterSchedule
Filters a schedule according to its parameters, removing any *changes* that do not occur within `[start, end)` as well as doing tag filtering. Always returns at least the last prompt in the schedule if everything would otherwise be filtered.

`start=0, end=0` returns the prompt at the start and `start=1.0, end=1.0` returns the prompt at the end.

## ScheduleToCond
Produces a combined conditioning for the appropriate timesteps. From a schedule. Also applies LoRAs to the CLIP model according to the schedule.

## ScheduleToModel
Produces a model that'll cause the sampler to reapply LoRAs at specific steps according to the schedule.

This depends on a callback handled by a monkeypatch of the ComfyUI sampler function, so it might not work with custom samplers, but it shouldn't interfere with them either.

## PCSplitSampling
Causes sampling to be split into multiple sampler calls instead of relying on timesteps for scheduling. This makes the schedules more accurate, but seems to cause weird behaviour with SDE samplers. (Upstream bug?)

## Older nodes

- `EditableCLIPEncode`: A combination of `PromptToSchedule` and `ScheduleToCond`
- `LoRAScheduler`: A combination of `PromptToSchedule`, `FilterSchedule` and `ScheduleToModel`

# Known issues

- `CUT` does not work with `STYLE:perp`
- `PCSplitSampling` overrides ComfyUI's `BrownianTreeNoiseSampler` noise sampling behaviour so that each split segment doesn't add crazy amounts of noise to the result with some samplers.
- Split sampling may have weird behaviour if your step percentages go below 1 step.
- Interpolation is probably buggy and will likely change behaviour whenever code gets refactored.
- If execution is interrupted and LoRA scheduling is used, your models might be left in an undefined state until you restart ComfyUI
