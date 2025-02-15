# Approximate Training
:label:`sec_approx_train`

Recall what we have discussed in :numref:`sec_word2vec`.
The main idea of the skip-gram model is 
using softmax operations to calculate
the conditional probability of
generating a context word $w_o$
based on the given center word $w_c$:

$$P(w_o \mid w_c) = \frac{\text{exp}(\mathbf{u}_o^\top \mathbf{v}_c)}{ \sum_{i \in \mathcal{V}} \text{exp}(\mathbf{u}_i^\top \mathbf{v}_c)},$$

whose corresponding logarithmic loss is given by

$$-\log P(w_o \mid w_c) = -\mathbf{u}_o^\top \mathbf{v}_c + \log\left(\sum_{i \in \mathcal{V}} \text{exp}(\mathbf{u}_i^\top \mathbf{v}_c)\right).$$
:eqlabel:`eq_approx-training-log-skipgram`


Due to the nature of the softmax operation,
since a context word may be anyone in the 
dictionary $\mathcal{V}$,
:eqref:`eq_approx-training-log-skipgram`
contains the summation 
of items as many as the entire size of the vocabulary.
As a result,
the gradient calculation
for the skip-gram model
in :eqref:`skip-gram-grad`
and that
for the continuous bag-of-words model
in :eqref:`eq_cbow-gradient`
both contain
the "big" summation.
For a large dictionary with
hundreds of thousands or millions of words, 
the computational cost 
for calculating such gradients
is huge.

In order to reduce the aforementioned computational complexity, this section will introduce two approximate training methods: 
*negative sampling* and *hierarchical softmax*.
Due to the similarity
between the skip-gram model and
the continuous bag of words model, 
we will just take the skip-gram model as an example
to describe these two approximate training methods.



## Negative Sampling
:label:`subsec_negative-sampling`

Negative sampling modifies the original objective function. Given a context window for the center word $w_c$, we will treat it as an event for context word $w_o$ to appear in the context window and compute the probability of this event from

$$P(D=1\mid w_c, w_o) = \sigma(\mathbf{u}_o^\top \mathbf{v}_c),$$

Here, the $\sigma$ function has the same definition as the sigmoid activation function:

$$\sigma(x) = \frac{1}{1+\exp(-x)}.$$

We will first consider training the word vector by maximizing the joint probability of all events in the text sequence. Given a text sequence of length $T$, we assume that the word at time step $t$ is $w^{(t)}$ and the context window size is $m$. Now we consider maximizing the joint probability

$$ \prod_{t=1}^{T} \prod_{-m \leq j \leq m,\ j \neq 0} P(D=1\mid w^{(t)}, w^{(t+j)}).$$

However, the events included in the model only consider positive examples. In this case, only when all the word vectors are equal and their values approach infinity can the joint probability above be maximized to 1. Obviously, such word vectors are meaningless. Negative sampling makes the objective function more meaningful by sampling with an addition of negative examples. Assume that event $P$ occurs when context word $w_o$ appears in the context window of center word $w_c$, and we sample $K$ words that do not appear in the context window according to the distribution $P(w)$ to act as noise words. We assume the event for noise word $w_k$($k=1, \ldots, K$) to not appear in the context window of center word $w_c$ is $N_k$. Suppose that events $P$ and $N_1, \ldots, N_K$ for both positive and negative examples are independent of each other. By considering negative sampling, we can rewrite the joint probability above, which only considers the positive examples, as

$$ \prod_{t=1}^{T} \prod_{-m \leq j \leq m,\ j \neq 0} P(w^{(t+j)} \mid w^{(t)}),$$

Here, the conditional probability is approximated to be
$$ P(w^{(t+j)} \mid w^{(t)}) =P(D=1\mid w^{(t)}, w^{(t+j)})\prod_{k=1,\ w_k \sim P(w)}^K P(D=0\mid w^{(t)}, w_k).$$


Let the text sequence index of word $w^{(t)}$ at time step $t$ be $i_t$ and $h_k$ for noise word $w_k$ in the dictionary. The logarithmic loss for the conditional probability above is

$$
\begin{aligned}
-\log P(w^{(t+j)} \mid w^{(t)})
=& -\log P(D=1\mid w^{(t)}, w^{(t+j)}) - \sum_{k=1,\ w_k \sim P(w)}^K \log P(D=0\mid w^{(t)}, w_k)\\
=&-  \log\, \sigma\left(\mathbf{u}_{i_{t+j}}^\top \mathbf{v}_{i_t}\right) - \sum_{k=1,\ w_k \sim P(w)}^K \log\left(1-\sigma\left(\mathbf{u}_{h_k}^\top \mathbf{v}_{i_t}\right)\right)\\
=&-  \log\, \sigma\left(\mathbf{u}_{i_{t+j}}^\top \mathbf{v}_{i_t}\right) - \sum_{k=1,\ w_k \sim P(w)}^K \log\sigma\left(-\mathbf{u}_{h_k}^\top \mathbf{v}_{i_t}\right).
\end{aligned}
$$

Here, the gradient computation in each step of the training is no longer related to the dictionary size, but linearly related to $K$. When $K$ takes a smaller constant, the negative sampling has a lower computational overhead for each step.


## Hierarchical Softmax

Hierarchical softmax is another type of approximate training method. It uses a binary tree for data structure as illustrated in :numref:`fig_hi_softmax`, with the leaf nodes of the tree representing every word in the dictionary $\mathcal{V}$.

![Hierarchical Softmax. Each leaf node of the tree represents a word in the dictionary. ](../img/hi-softmax.svg)
:label:`fig_hi_softmax`

We assume that $L(w)$ is the number of nodes on the path (including the root and leaf nodes) from the root node of the binary tree to the leaf node of word $w$. Let $n(w, j)$ be the $j^\mathrm{th}$ node on this path, with the context word vector $\mathbf{u}_{n(w, j)}$. We use :numref:`fig_hi_softmax` as an example, so $L(w_3) = 4$. Hierarchical softmax will approximate the conditional probability in the skip-gram model as

$$P(w_o \mid w_c) = \prod_{j=1}^{L(w_o)-1} \sigma\left( [\![  n(w_o, j+1) = \text{leftChild}(n(w_o, j)) ]\!] \cdot \mathbf{u}_{n(w_o, j)}^\top \mathbf{v}_c\right),$$

Here the $\sigma$ function has the same definition as the sigmoid activation function, and $\text{leftChild}(n)$ is the left child node of node $n$. If $x$ is true, $[\![x]\!] = 1$; otherwise $[\![x]\!] = -1$.
Now, we will compute the conditional probability of generating word $w_3$ based on the given word $w_c$ in :numref:`fig_hi_softmax`. We need to find the inner product of word vector $\mathbf{v}_c$ (for word $w_c$) and each non-leaf node vector on the path from the root node to $w_3$. Because, in the binary tree, the path from the root node to leaf node $w_3$ needs to be traversed left, right, and left again (the path with the bold line in :numref:`fig_hi_softmax`), we get

$$P(w_3 \mid w_c) = \sigma(\mathbf{u}_{n(w_3, 1)}^\top \mathbf{v}_c) \cdot \sigma(-\mathbf{u}_{n(w_3, 2)}^\top \mathbf{v}_c) \cdot \sigma(\mathbf{u}_{n(w_3, 3)}^\top \mathbf{v}_c).$$

Because $\sigma(x)+\sigma(-x) = 1$, the condition that the sum of the conditional probability of any word generated based on the given center word $w_c$ in dictionary $\mathcal{V}$ be 1 will also suffice:

$$\sum_{w \in \mathcal{V}} P(w \mid w_c) = 1.$$

In addition, because the order of magnitude for $L(w_o)-1$ is $\mathcal{O}(\text{log}_2|\mathcal{V}|)$, when the size of dictionary $\mathcal{V}$ is large, the computational overhead for each step in the hierarchical softmax training is greatly reduced compared to situations where we do not use approximate training.

## Summary

* Negative sampling constructs the loss function by considering independent events that contain both positive and negative examples. The gradient computational overhead for each step in the training process is linearly related to the number of noise words we sample.
* Hierarchical softmax uses a binary tree and constructs the loss function based on the path from the root node to the leaf node. The gradient computational overhead for each step in the training process is related to the logarithm of the dictionary size.

## Exercises

1. Before reading the next section, think about how we should sample noise words in negative sampling.
1. What makes the last formula in this section hold?
1. How can we apply negative sampling and hierarchical softmax in the skip-gram model?

[Discussions](https://discuss.d2l.ai/t/382)
