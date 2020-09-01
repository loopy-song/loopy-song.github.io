---
layout: post
title: On Implementing Smoothing Techniques for Markov Password Models
math: true
---

**If you are familiar with how to implement smoothing algorithms, just skip this post.**

*Before we begin, let's make a few things clear. By "Markov models", I mean n-gram models here. Also, smoothing can mean a lot of different things, like in [developer notes of PCFG cracker](https://github.com/lakiw/pcfg_cracker/wiki/Developer-Notes) it roughly means quantization; in this post, smoothing has its natural meaning in n-gram models, that is, assigning some probabilities to unseen events so as to improve the accuracy of the model. Each password is by default padded with one or more start symbols and one end symbol.*

It's Sept 2020 now; why talk about smoothing, when there are already excellent works like "[A Study of Probabilistic Password Models](https://www.ieee-security.org/TC/SP2014/papers/AStudyofProbabilisticPasswordModels.pdf)" and "[Fast, Lean, and Accurate: Modeling Password Guessability Using Neural Networks](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/melicher)"? Markov models are still important today, as they are much faster than RNN/GAN-based models in password cracking, where speed matters. However, there are few implementations of Markov password models publicly available, and even fewer support smoothing. So when you try to do smoothing, you often have to write one yourself. This can be particularly tricky if you want to implement some intricate smoothing techniques like Katz backoff or modified Kneser-Ney, as you cannot easily verify whether your code is consistent with these models or not. Besides, I find it difficult to implement smoothing techniques solely by descriptions from password papers, which often have to omit some details due to space constraints.

I have implemented some of the smoothing techniques for password cracking, but I'm not skilled enough to write a working password module. Instead, I want to talk about how to implement these smoothing techniques correctly, and share some demo code (when I finish cleaning it). If you are new to smoothing techniques for Markov password models, see "[A Study of Probabilistic Password Models](https://www.ieee-security.org/TC/SP2014/papers/AStudyofProbabilisticPasswordModels.pdf)" by Ma et al. or resources section.

<!-- more -->

{% raw %}

## Laplace Smoothing

Laplace (addictive, add-one, add-theta) smoothing is not hard to implement, so I'll skip it here.

## Good-Turing Smoothing

As for Good-Turing (GT) smoothing, Ma et al. have shown that "Good-Turing smoothing underperforms Laplace smoothing, and results in significant overfitting for order 5". But if you are still interested, read on.

Well, Good-Turing estimate itself is relatively easy, but what are we applying the estimation over? This really puzzles me, so I'll just write down several possibilities here. If you have any idea about this, please tell me. I think we may apply GT estimation over:

- all passwords. This one seems more coherent with Ma et al.'s description, but you may have to use combinatorial methods to get n-gram counts.
- all n-grams. This one is more commonly used in language modeling, but it still underperforms Laplace smoothing.
- n-grams sharing the same histories. This one suffers greatly from sparsity, but it works better than the first two if you allow it to degenerate to Laplace smoothing sometimes.

Simple Good-Turing (SGT) smoothing is a lot like Good-Turing smoothing, and can be implemented in similar ways. I suggest that you stick to [this code](https://www.grsampson.net/D_SGT.c) by Geoffrey Sampson, because it's an "official" version of SGT, and other descriptions may slightly differ from Gale and Sampson's original paper. For example, equation 5.3 in "Guessing human-chosen secrets" is different from the original paper in $$m = \max(m)$$ case (but it doesn't really matter).

## Backoff

The backoff model I talk about here is the one described by Ma et al., i.e., Katz's backoff without Good-Turing discounting. It should be easy to plug in Good-Turing discounting if you want. If you prefer a mathematical definition, see this [Wikipedia page](https://en.wikipedia.org/wiki/Katz's_back-off_model) on Katz's backoff. To make my words less obscure I'll first copy the formulas from Wikipedia and paste them here (removing GT discounting for now):

$$
{\begin{aligned}&P_{{bo}}(w_{i}\mid w_{{i-n+1}}\ldots w_{{i-1}})\\[4pt]={}&{\begin{cases}{\dfrac  {C(w_{{i-n+1}}\ldots w_{{i-1}}w_{{i}})}{C(w_{{i-n+1}}\ldots w_{{i-1}})}}&{\text{if }}C(w_{{i-n+1}}\ldots w_{i})>k\\[10pt]\alpha _{{w_{{i-n+1}}\ldots w_{{i-1}}}}P_{{bo}}(w_{i}\mid w_{{i-n+2}}\ldots w_{{i-1}})&{\text{otherwise.}}\end{cases}}\end{aligned}}
$$

To compute $$\alpha$$, it is useful to first define a quantity $$\beta$$, which is the left-over probability mass for the (n−1)-gram:

$$
\beta _{{w_{{i-n+1}}\ldots w_{{i-1}}}}=1-\sum _{{\{w_{i}:C(w_{{i-n+1}}\ldots w_{{i}})>k\}}}{\frac  {C(w_{{i-n+1}}\ldots w_{{i-1}}w_{{i}})}{C(w_{{i-n+1}}\ldots w_{{i-1}})}}.
$$

Then the back-off weight, $$\alpha$$, is computed as follows:

$$
\alpha _{{w_{{i-n+1}}\ldots w_{{i-1}}}}={\frac  {\beta _{{w_{{i-n+1}}\ldots w_{{i-1}}}}}{\sum _{{\{w_{i}:C(w_{{i-n+1}}\ldots w_{{i}})\leq k\}}}P_{{bo}}(w_{i}\mid w_{{i-n+2}}\ldots w_{{i-1}})}}.
$$

The backoff weight $$\alpha$$ is quite important here, as it seems natural to use $$\beta$$ as backoff weight, which will leave the model un-normalized. Also, care should be taken when sampling from the backoff model, especially if you do not precompute probabilities.

Repo [montecarlopwd](https://github.com/matteodellamico/montecarlopwd) by Dell'Amico is a very good reference if you want to implement backoff model. It supports both pre-computing probabilities as well as on-the-fly computation. You can check out the paper "[Monte Carlo Strength Evaluation: Fast and Reliable Password Checking](http://www.eurecom.fr/~filippon/Publications/ccs15.pdf)" for implementation details. I think the original implementation is a bit inconsistent with the backoff model we discussed here, but it's not clear which one works better. I made some changes in my forked repo to make it match Ma et al.'s description. 

Ma et al. reported that "The backoff models used in experimentation are approximately 11 times slower than the plain Markov models, both for password generation and for probability estimation", but if implemented carefully the overhead of backoff model won't be very large. Specifically, if you want to compute probabilities on-the-fly, don't try to get backoff weight $$\alpha$$ directly using the above formula, or you'll fall into the big hole of recursion. Instead, calculate $$\alpha$$ like this:

$$
{\begin{aligned}\alpha _{{w_{{i-n+1}}\ldots w_{{i-1}}}}&={\frac  {\beta _{{w_{{i-n+1}}\ldots w_{{i-1}}}}}{\sum _{{\{w_{i}:C(w_{{i-n+1}}\ldots w_{{i}})\leq k\}}}P_{{bo}}(w_{i}\mid w_{{i-n+2}}\ldots w_{{i-1}})}}\\&= {\frac  {\beta _{{w_{{i-n+1}}\ldots w_{{i-1}}}}}{1 - \sum _{{\{w_{i}:C(w_{{i-n+1}}\ldots w_{{i}})> k\}}}P_{{bo}}(w_{i}\mid w_{{i-n+2}}\ldots w_{{i-1}})}}\\ &= {\frac  {\beta _{{w_{{i-n+1}}\ldots w_{{i-1}}}}}{1-\sum _{{\{w_{i}:C(w_{{i-n+1}}\ldots w_{{i}})> k\}}}\frac{C(w_{i-n+2}\ldots w_{i-1}w_i)}{C(w_{i-n+2}\ldots w_{i-1})}}}.\end{aligned}}
$$

The last equality holds because $$C(w_{i-n+2} \ldots w_i) \geq C(w_{i-n+1} \ldots w_i) > k$$. Now we've got a formula for backoff weight without any recursion. Backoff model with precomputation can help you avoid such trouble, and is faster in testing phase, but it is not so memory-efficient and brings some overhead in training.

## Kneser-Ney Smoothing

Modified Kneser-Ney (MKN) smoothing is widely used in language modeling, but to my knowledge, there's no public work that uses any kind of Kneser-Ney smoothing in password guessing. I've implemented MKN smoothing and found that it performs quite well. One benefit of MKN smoothing is that, like other interpolation methods, MKN enables you to use Markov models of larger order (like 6, 8, or 10) without having to worry about sparsity (but you should definitely worry about memory usage). I observe a 0.02-0.05 raise (about 10% relatively) in cracking rate compared to other Markov models with 10^7 guesses.

MKN is rather complicated and I just don't understand it well even now. I think it's helpful to see Kneser-Ney smoothing as a approximate inference scheme in the hierarchical Pitman-Yor language model, as in "[A Bayesian Interpretation of Interpolated Kneser-Ney](http://www.stats.ox.ac.uk/~teh/research/compling/hpylm.pdf)" by Teh.

I cannot find an implementation of MKN that is both correct and simple, but [this repo](https://github.com/smilli/kneser-ney) is very simple and almost correct, so it should serve as a good starting point. Note how Kneser-Ney uses adjusted counts instead of real counts, and how probabilities are computed and interpolated. To fix it you may have to modify function  `_calc_unigram_probs` (interpolate with uniform distribution, not important) and `logprob` (multiply/add backoff factor when backing off, important). If you spot a better one, please tell me. And you can always try to modify language toolkits to suit your needs in password guessing if you have strong coding skills. I'll release a demo of MKN that works faster when I finish cleaning it, so I'll talk about more implementation details then.

It's possible that other Kneser-Ney variants outperform MKN in password guessing; for example, you can try to use more discounts instead of 3, or use different methods to calculate discounts. I tried a few of them and see nothing promising, but maybe you can find a better variant if you try! Anyway, I hope that Kneser-Ney smoothing can be accepted and used (for legitimate purposes, of course) by password community in the future. Also, if you're working on an open-source project on smoothing for password models, please let me know and I'd be happy to contribute.

{% endraw %}

## Resources

If you are not yet familiar with n-gram models, smoothing, or their use in password security, here are something that might be useful.

### Papers

For a comparison of different n-gram language models and an introduction of modified Kneser-Ney smoothing, see [Chen and Goodman '98](https://www.cs.ubc.ca/~carenini/TEACHING/CPSC503-04/chen-goodman-tr-10-98.pdf) or [Chen and Goodman '99](http://u.cs.biu.ac.il/~yogo/courses/mt2013/papers/chen-goodman-99.pdf).

For publications on Markov-based password guessing, check out readme file of [NEMO](https://github.com/RUB-SysSec/NEMO) for a good overview, and below are some recent pubs not yet included in it: 

- Wang et al., [Birthday, Name and Bifacial-security: Understanding Passwords of Chinese Web Users](https://www.usenix.org/system/files/sec19-wang-ding.pdf), in Usenix Security '19.

### Slides

These slides can serve as good introductions to n-gram language models:

- Bill MacCartney, [NLP Lunch Tutorial: Smoothing](https://nlp.stanford.edu/~wcmac/papers/20050421-smoothing-tutorial.pdf)
- Philipp Koehn, [Chapter 7: Language Models](http://www.statmt.org/book/slides/07-language-models.pdf)
- Dan Klein, [Lecture 2: Language Models](https://people.eecs.berkeley.edu/~klein/cs288/sp10/slides/SP10 cs288 lecture 2 -- language models (2PP).pdf)
- Dan Jurafsky, [Language Modeling](https://web.stanford.edu/class/cs124/lec/languagemodeling.pdf)

### Book Chapter

Check out [Chapter 3: N-gram Language Models](https://web.stanford.edu/~jurafsky/slp3/) of *Speech and Language Processing, 3rd edn.* by Daniel Jurafsky & James H. Martin. (You might've read it already, though.)

### Code

Here are several projects that use (smoothed) Markov models for password guessing and/or password strength estimation:

- [montecarlopwd](https://github.com/matteodellamico/montecarlopwd) features an implementation (two impls, actually) of backoff model;
- [OMEN](https://github.com/RUB-SysSec/OMEN) is a very fast Markov model-based password guesser, and supports addictive smoothing;
- [NEMO](https://github.com/RUB-SysSec/NEMO) uses Markov models to measure password strength, and supports addictive smoothing;
- [PGS](https://pgs.ece.cmu.edu/), or Password Guessability Service, supports Laplace smoothed Markov model.
