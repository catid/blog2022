+++
title = "Low-Rank Approximations to Linear Layers Don't Work"
date = "2024-02-05"
cover = "posts/low_rank/low_go.webp"
description = "There is no free lunch"
+++


## Low-Rank Approximations to Linear Layers Don't Work

Recently I've been looking at ways to slim down transformer models so that I can train larger models at home.  For the most part I have not had success with low-rank approximation methods, where you replace one of the `nn.Linear` layers in the model with a low-rank approximation.

You can find papers out there that seem to be able to train models with low-rank linear layers, but I don't think they're correct.  In this blog post I'll try to reproduce one of these results to demonstrate this phenomenon.  If they really did work we'd be using sparse or low-rank matrices everywhere in machine learning - and of course we are not.

To be clear, pruning and quantization and other sparsification approaches *after* training are totally legitimate and are commonly used.  But we're talking about training with low-rank or sparse matrices here.


## The Basics

Rank reduction after training has been applied successfully to *improve* benchmark scores after training very large models: https://github.com/pratyushasharma/laser

However I have yet to see it improve accuracy in a smaller model.  I tried implementing some ideas from the LASER paper and did not see any clear benefit aside from one case: Reducing the rank of the MLP input projection on the final layer.  But, it was a marginal improvement if anything.  Not worth the effort since it doesn't change the model size or inference speed.

When I was testing that idea I used this code to replace the linear layer with a low-rank approximation:

```python
# Rank reduction layer: Reduce number of parameters by a given factor e.g. 0.5 = 50% smaller model.
class RankReductionLayer(nn.Module):
    def __init__(self, d_in, d_out, r=0.5):
        super(RankReductionLayer, self).__init__()
        rank = int(r * d_in * d_out / (d_in + d_out))

        self.down = nn.Linear(d_in, rank)
        self.up = nn.Linear(rank, d_out)
    
    def forward(self, x):
        return self.up(self.down(x))
```

To be clear this is a drop-in replacement for a `nn.Linear` layer that is designed to cut the number of parameters in half.


## A New Contender

Today, I found a new idea that seemed promising at first, and indeed does better than the low-rank linear layer above: Stacked Kronecker-product Layers https://openreview.net/pdf?id=ZjGr1tMVbjw

The main downside is that they slow down training and inference, though in theory they might be able to reduce the number of parameters in the model without hurting model accuracy.  Let's find out!  First, here's my implementation of the layer:

```python
def round_up_sqrt(m):
    sm = int(m ** 0.5 + 0.5)
    while sm*sm > m:
        sm -= 1
    while sm*sm < m:
        sm += 1
    return sm

# Stacked Kronecker-product Layers https://openreview.net/pdf?id=ZjGr1tMVbjw
# Uses 2r*sqrt(nm) parameters instead of nm.
# For for n=512 x m=2048, r must be 256 or less to make it worthwhile.
class SKLinear(nn.Module):
    def __init__(self, n, m, scale=0.5):
        super().__init__()
        self.n = n
        self.m = m
        self.sn = round_up_sqrt(n)
        self.np = self.sn * self.sn # n rounded up to next square
        self.sm = round_up_sqrt(m)
        self.mp = self.sm * self.sm # m rounded up to next square
        k = self.sn * self.sm

        r = int(scale * (n * m) / 2.0 / k + 0.5)

        print(f"Using SKLinear: n={n} m={m} r={r} round_sqrt(n)={self.sn} round_sqrt(m)={self.sm} n'={self.np} m'={self.mp} k={k} reduction={(2 * r * k) * 100.0 / (n * m)}%")

        # Initialize A and B using Kaiming initialization
        self.A = nn.Parameter(torch.empty(k, r))
        nn.init.kaiming_uniform_(self.A, a=math.sqrt(5)) # a is the parameter for the ReLU
        self.B = nn.Parameter(torch.empty(r, k))
        nn.init.kaiming_uniform_(self.B, a=math.sqrt(5))

    def forward(self, x):
        # Validate that the inputs are of the expected sizes
        if x.size(-1) != self.n:
            raise ValueError("Input vector must have size n")

        S = torch.matmul(self.A, self.B).reshape(self.sn, self.sm, self.sn, self.sm).transpose(1, 2).reshape(self.np, self.mp)
        S_truncated = S[:self.n, :self.m]

        return torch.matmul(x, S_truncated)
```

Note that the `kaiming_uniform_` init is important for FP16 training.

As you can see, this implementation is parameterized by the number of weights in the model as compared to a full linear layer.  So we can directly compare these two layers to eachother and to e.g. reducing the hidden dimension of the FFN layer by 50% and see which one does better.

I modified the `x-transformers` library to use this new layer to replace different linear layers in the model, and tested it on CIFAR-10.  The results are below.


## CIFAR-10 Validation Accuracy% Results

Results from training using my CIFAR-10 scripts: https://github.com/catid/cifar10deepspeed and running `./launch_local_train.sh --reset --arch x_transformers`

```
(A) CIFAR-10 x-transformers baseline: 88.6%
(B) Set FFN hidden_dim to be half size: 88.39%

(C) Replace FFN-in+FFN-out with SKL*(r=0.5): 87.32% 

Note: SKL = Stacked Kronecker-product Layer (the complicated new one)

(D) Replace FFN-in with SKL(r=0.5): 88.29%
(E) Replace FFN-in with simple low-rank(r=0.5): 87.83% 

(F) Replace FFN-out with SKL(r=0.5): 88.24% 

(G) Replace Attn-out with SKL(r=0.5): 88.77%
(G) Repeated training: 88.54%

(H) Replace Attn-out with simple low-rank(r=0.5): 88.09% 

(I) Replace all attention with SKL(r=0.5): 87.47% 
```

As you can see from the validation accuracy% results, the original unmodified network produces the best results as you would expect (A).

The simple low-rank linear layers perform worse than this new layer type by far.

Also as I expected (B), reducing the hidden_dim of the FFN is a far better way to reduce the number of parameters in the model than using low-rank approximations.  For one, it actually reduces the training/inference time.  The low-rank approximations do not improve the speed as much (simple low-rank layer) or make it worse (Kronecker layer).

The (G) modification reduces parameters (from 5812538 to 5606202 weights) by just 4%.  And it's really questionable if there is any difference from baseline performance here.  It could be 4% worse - I can't tell since the difference in results are in the noise.


## Conclusion

So yet again, despite me wanting it to work, it doesn't.  I'll probably keep tilting at this windmill because it's a very compelling idea.  Like maybe it works on much larger models I haven't tried training yet.  Hope springs eternal.


## Follow-up: DYAD

Euclaise suggested trying DYAD ( https://arxiv.org/pdf/2312.06881.pdf ) to see how it performs during pre-training.

I replaced the FeedForward linear layers in ViT-tiny with DYAD, and trained on CIFAR-10.  This is a different model than the x-transformers model I used above.

Hyperparameters:

```
* ViT-tiny Patch Size = 4x4
* ViT-tiny dim=512
* ViT-tiny depth=4
* ViT-tiny heads=6
* ViT-tiny mlp_dim=256
* Microbatch size per GPU = 256 (2x 4090 GPUs)
* Gradient accumulation steps = 2
* FP16 training
* Optimizer = AdamW
* Learning rate = 9.18e-4
* Weight decay = 1.5e-5
* Max epochs = 300
* Warmup epochs = 5
* Scheduler = CosineAnnealingWarmRestarts
* Code here: https://github.com/catid/cifar10deepspeed
```

Baseline results:

```
Parameters: 4363254
Validation accuracy: 85.46%, 84.96%, 85.76%, 85.48%
```

Results from increasing mlp_dim=1024, and replacing FFN linear layers with DYAD (which has 4x fewer parameters):

```
Parameters: 4366326
Validation accuracy: 85.14%, 84.67%, 84.69%, 84.88%
```

Summary: DYAD performs about 1% worse for the same number of parameters.  This is about the same as the SKL results above.  Note that DYAD also reduces training speed by about 23%.

So why do these low-rank approximations perform worse for the same number of parameters?  They are like the full linear layers, but they have fewer connections between the parameters.  There's something about pruning connections between parameters before training starts that doesn't work.

My theory is that by having fewer connections, there are more gradient bottlenecks in the network.  This points at a way to speed up training as well: What if we start with more connections in the network and over time prune them?  What if we start with a wide network with lots of connections, and end up with a deeper, but narrower network?  This seems like a better way to do pre-training.
