# minGPT-tuned

This a tuned minGPT with more tricks and tweaks, while preserving the simplicity of the original project.

## Performance

Data: https://github.com/karpathy/char-rnn/blob/master/data/tinyshakespeare/input.txt

Model: block_size = 128, n_layer=3, n_head=8, n_embd=512 [we use a smaller model to test its capacity]

Trainer: max_epochs=150, batch_size=256, learning_rate=6e-4, lr_decay=True, warmup_tokens=512 * 20, final_tokens=150 * len(train_dataset) * block_size

minGPT-tuned:
```
number of parameters: 9.983488e+06
epoch 1 iter 33: train loss 2.61996. lr 5.999354e-04:  [00:08<00:00,  4.19it/s]
epoch 2 iter 33: train loss 2.22801. lr 5.997392e-04:  [00:07<00:00,  4.35it/s]
epoch 3 iter 33: train loss 2.06643. lr 5.994116e-04:  [00:07<00:00,  4.36it/s]
......
epoch 50 iter 33: train loss 1.00424. lr 4.500336e-04:  [00:07<00:00,  4.31it/s]
......
epoch 100 iter 33: train loss 0.71006. lr 1.500168e-04:  [00:07<00:00,  4.31it/s]
......
epoch 148 iter 33: train loss 0.60897. lr 6.000000e-05:  [00:07<00:00,  4.32it/s]
epoch 149 iter 33: train loss 0.61404. lr 6.000000e-05:  [00:07<00:00,  4.31it/s]
epoch 150 iter 33: train loss 0.60880. lr 6.000000e-05:  [00:07<00:00,  4.32it/s]
```

minGPT:
```
number of parameters: 9.590272e+06
epoch 1 iter 33: train loss 2.64701. lr 5.999354e-04:  [00:07<00:00,  4.39it/s]
epoch 2 iter 33: train loss 2.49893. lr 5.997392e-04:  [00:07<00:00,  4.62it/s]
epoch 3 iter 33: train loss 2.43154. lr 5.994116e-04:  [00:07<00:00,  4.60it/s]
......
epoch 50 iter 33: train loss 1.08477. lr 4.500336e-04:  [00:07<00:00,  4.55it/s]
......
epoch 100 iter 33: train loss 0.80659. lr 1.500168e-04:  [00:07<00:00,  4.57it/s]
......
epoch 148 iter 33: train loss 0.67977. lr 6.000000e-05:  [00:07<00:00,  4.57it/s]
epoch 149 iter 33: train loss 0.69573. lr 6.000000e-05:  [00:07<00:00,  4.57it/s]
epoch 150 iter 33: train loss 0.68462. lr 6.000000e-05:  [00:07<00:00,  4.51it/s]
```

## Trick 1 : Time-weighting (there might be similar ideas in other papers)

```python
self.time_weighting = nn.Parameter(torch.ones(self.n_head, config.window_len, config.window_len))
......
att = F.softmax(att, dim=-1)
att = att * self.time_weighting[:,:T,:T] # this is "time-weighting"
att = self.attn_drop(att)
```

Time-weighting works, because tokens from different distances shall have different impacts on the current token.

Moreover, the self-attention effects shall be reduced for earlier tokens because they have shorter history-windows.

p.s. there might be a closed-form solution for optimal time-weighting. 

![time-weighting](time-weighting.jpg)

## Trick 2 : Time-mixing (this might be a new idea first shown here)

```python
self.time_shift = nn.ZeroPad2d((0,0,1,0))
......
x = torch.cat([self.time_shift(x)[:,:T,:C//2], x[:,:T,C//2:]], dim=2) # this is "time-mixing"
q = self.query(x).view(B, T, self.n_head, C // self.n_head).transpose(1, 2)
k = self.key(x).view(B, T, self.n_head, C // self.n_head).transpose(1, 2)
v = self.value(x).view(B, T, self.n_head, C // self.n_head).transpose(1, 2)
```

Time-mixing is a strange-looking operation designed by me (after pondering on how self-attention works).

Time-mixing enables more interesting local-attention effects, and makes the model directly capable of learning an essential aspect of human language: if we saw "AX" and then we see "A", we are more likely to expect an "X" after "A" (in the Bayesian sense).

p.s. you can try interweaving time-mixing channels in different layers.

## Troubleshooting

- Reduce batch_size if you encounter out-of-video-memory. 
- Set num_workers to 0 if you encounter pipe errors.
- Remember to change final_tokens when you change max_epochs.

---

# minGPT

![mingpt](mingpt.jpg)

A PyTorch re-implementation of [GPT](https://github.com/openai/gpt-3) training. minGPT tries to be small, clean, interpretable and educational, as most of the currently available ones are a bit sprawling. GPT is not a complicated model and this implementation is appropriately about 300 lines of code, including boilerplate and a totally unnecessary custom causal self-attention module. Anyway, all that's going on is that a sequence of indices goes into a sequence of transformer blocks, and a probability distribution of the next index comes out. The rest of the complexity is just being clever with batching (both across examples and over sequence length) so that training is efficient.

The core minGPT "library" (hah) is two files: `mingpt/model.py` contains the actual Transformer model definition and `mingpt/trainer.py` is (GPT-independent) PyTorch boilerplate that trains the model. The attached Jupyter notebooks then show how the "library" (hah) can be used to train sequence models:

- `play_math.ipynb` trains a GPT focused on addition (inspired by the addition section in the GPT-3 paper)
- `play_char.ipynb` trains a GPT to be a character-level language model on arbitrary text, similar to my older char-rnn but with a transformer instead of an RNN
- `play_words.ipynb` a BPE version that does not yet exist

With a bpe encoder, distributed training and maybe fp16 this implementation may be able to reproduce GPT-1/GPT-2 results, though I haven't tried $$$. GPT-3 is likely out of reach as my understanding is that it does not fit into GPU memory and requires a more careful model-parallel treatment.

### Example usage

This code is simple enough to just hack inline, not "used", but current API looks something like:

```python

# you're on your own to define a class that returns individual examples as PyTorch LongTensors
from torch.utils.data import Dataset
train_dataset = MyDataset(...)
test_dataset = MyDataset(...)

# construct a GPT model
from mingpt.model import GPT, GPTConfig
mconf = GPTConfig(vocab_size, block_size, n_layer=12, n_head=12, n_embd=768) # a GPT-1
model = GPT(mconf)

# construct a trainer
from mingpt.trainer import Trainer, TrainerConfig
tconf = TrainerConfig(max_epochs=10, batch_size=256)
trainer = Trainer(model, train_dataset, test_dataset, tconf)
trainer.train()
# (... enjoy the show for a while... )

# sample from the model (the [None, ...] and [0] are to push/pop a needed dummy batch dimension)
from mingpt.utils import sample
x = torch.tensor([1, 2, 3], dtype=torch.long)[None, ...] # context conditioning
y = sample(model, x, steps=30, temperature=1.0, sample=True, top_k=5)[0]
print(y) # our model filled in the integer sequence with 30 additional likely integers
```

### References

Code:

- [openai/gpt-2](https://github.com/openai/gpt-2) has the model but not the training code, and in TensorFlow
- [openai/image-gpt](https://github.com/openai/image-gpt) has some more modern gpt-3 like modification in its code, good reference as well
- huggingface/transformers has a [language-modeling example](https://github.com/huggingface/transformers/tree/master/examples/language-modeling). It is full-featured but as a result also somewhat challenging to trace. E.g. some large functions have as much as 90% unused code behind various branching statements that is unused in the default setting of simple language modeling.

Papers + some implementation notes:

#### Improving Language Understanding by Generative Pre-Training (GPT-1)

- Our model largely follows the original transformer work
- We trained a 12-layer decoder-only transformer with masked self-attention heads (768 dimensional states and 12 attention heads). For the position-wise feed-forward networks, we used 3072 dimensional inner states.
- Adam max learning rate of 2.5e-4. (later GPT-3 for this model size uses 6e-4)
- LR decay: increased linearly from zero over the first 2000 updates and annealed to 0 using a cosine schedule
- We train for 100 epochs on minibatches of 64 randomly sampled, contiguous sequences of 512 tokens.
- Since layernorm is used extensively throughout the model, a simple weight initialization of N(0, 0.02) was sufficient
- bytepair encoding (BPE) vocabulary with 40,000 merges
- residual, embedding, and attention dropouts with a rate of 0.1 for regularization.
- modified version of L2 regularization proposed in (37), with w = 0.01 on all non bias or gain weights
- For the activation function, we used the Gaussian Error Linear Unit (GELU).
- We used learned position embeddings instead of the sinusoidal version proposed in the original work
- For finetuning: We add dropout to the classifier with a rate of 0.1. learning rate of 6.25e-5 and a batchsize of 32. 3 epochs. We use a linear learning rate decay schedule with warmup over 0.2% of training. λ was set to 0.5.
- GPT-1 model is 12 layers and d_model 768, ~117M params

#### Language Models are Unsupervised Multitask Learners (GPT-2)

- LayerNorm was moved to the input of each sub-block, similar to a pre-activation residual network
- an additional layer normalization was added after the final self-attention block.
- modified initialization which accounts for the accumulation on the residual path with model depth is used. We scale the weights of residual layers at initialization by a factor of 1/√N where N is the number of residual layers. (weird because in their released code i can only find a simple use of the old 0.02... in their release of image-gpt I found it used for c_proj, and even then only for attn, not for mlp. huh. https://github.com/openai/image-gpt/blob/master/src/model.py)
- the vocabulary is expanded to 50,257
- increase the context size from 512 to 1024 tokens
- larger batchsize of 512 is used
- GPT-2 used 48 layers and d_model 1600 (vs. original 12 layers and d_model 768). ~1.542B params

#### Language Models are Few-Shot Learners (GPT-3)

- GPT-3: 96 layers, 96 heads, with d_model of 12,288 (175B parameters).
- GPT-1-like: 12 layers, 12 heads, d_model 768 (125M)
- We use the same model and architecture as GPT-2, including the modified initialization, pre-normalization, and reversible tokenization described therein
- we use alternating dense and locally banded sparse attention patterns in the layers of the transformer, similar to the Sparse Transformer
- we always have the feedforward layer four times the size of the bottleneck layer, dff = 4 ∗ dmodel
- all models use a context window of nctx = 2048 tokens.
- Adam with β1 = 0.9, β2 = 0.95, and eps = 10−8
- All models use weight decay of 0.1 to provide a small amount of regularization. (NOTE: GPT-1 used 0.01 I believe, see above)
- clip the global norm of the gradient at 1.0
- Linear LR warmup over the first 375 million tokens. Then use cosine decay for learning rate down to 10% of its value, over 260 billion tokens.
- gradually increase the batch size linearly from a small value (32k tokens) to the full value over the first 4-12 billion tokens of training, depending on the model size.
- full 2048-sized time context window is always used, with a special END OF DOCUMENT token delimiter

### License

MIT
