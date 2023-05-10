# 1. Arithmetization overhaul.

## Preliminaries (angry rant)

Before we go into actual proposals, we need to discuss some common misconceptions (stemming largely from the fact that Nova was proposed initially in R1CS arithmetization and Sangria was an attempt to replace it with PlonK).

I will refer to "Novaish" protocols, by which I actually mean a vague blob of protocols that operate in a following way:

1. We have a system of equations ("constraints") on a witness vector.
2. We make it homogeneous of degree $d$, where $d$ is a maximal possible degree occuring in the system; sparing only linear equations which we keep linear (remember copy constraints in Sangria?)
3. We use folding argument from Nova / Sangria, producing $d-1$ cross term, and obtain a folding scheme.
4. (Optional) we construct an IVC.
5. (Optional) we have a final SNARK to check the folded instance in one go without need to open.

Hypernova seems to be of a different kind and is probably a competitor to this framework itself - it seems to use its CCS arithmetization in a nontrivial way. I need to understand more of it to make my judgement on this (maybe the sumcheck argument is doable in different arithmetizations too).

Due to some sort of sad and extremely inconvenient misconception which I'm trying to dispel as hard as I can, the idea that arithmetization (specific form of equations) in (1) is somehow related to arithmetization in (5) became widespread.

In fact, it does not need to be that way (though, similarity can make the step (5) more efficient).

<details><summary><b>Let's give an example.</b></summary>

> Imagine that we are using Nova with a cycle of curves (Pallas/Vesta), and we want to use Halo2 IPA snark as the final check. In the end of the IVC process, we obtain a pair of instances, one over Pallas and other over Vesta, and need (in particular) to check that commitment to extended witness (relax.factor + witness + error-term) $t_{n+1}, [W_{n+1}], [E_{n+1}]$ opens to some actual vector $t_{n+1}, W_{n+1}, E_{n+1}$ satisfying our relaxed system of equations. <br><br> We want to use plonk. What we need to do is to give the vector $(W_{n+1}, E_{n+1})$ *as a column* to the PlonK, using *the same commitment key as we used for extended witness commitment for the halo2 vector commitment key*. This allows to move the data without decommitting it and committing it back; I will refer to this process as "passing a commitment" (and sometimes a more general process is called "fiat inputs", you can read about it [here](https://notes.ethereum.org/@dankrad/kzg_commitments_in_proofs)).<br><br> Next, we can just *emulate* the system of equations that W, E must satisfy inside of a PlonK circuit - this, of course, will have some overhead, but what's important is that we do not have to open a commitment. Which is nice.

</details>

To understand a bit more, let's imagine that we are using plonkish arithmetization already, so we are in the Sangria world. Our witness is, then, should, optimally, be mapped not into a single column, but into the collection of the columns! So, we do need some custom argument to cut the witness vector into parts. It is somewhat easy in KZG, and somewhat harder in IPA, my point is - even if our arithmetization *was* plonkish, it is not exactly clear how to map it into the final check arithmetization.

Therefore, I suggest we mentally and terminologically unbind them once and forever, for the common good.

## Proposed arithmetization for Novaish protocols

I suggest the following arithmetization:

1. We have a set of wires.
2. We have some equations on wire variables, of max degree $d$.
3. We have a special wire $t$ playing a role of constants - i.e. set up to $1$. //*this is analogous to Groth16 R1CS btw*

Now, we perform the process of homogenization for every *non-linear* equation in the set. I.e. if an equation has degree > 1, then every monomial of degree $k<d$ must be multiplied by $t^{d-k}$.

For every equation of degree $d>1$ we also introduce a RHS variable. Collection of all RHS variables is called *an error vector*.

<details><summary><b>Why linear eqs get the special treatment?</b></summary> 

> We can keep them exact and not relax them at all.</details>

<details><summary><b>Can't we homogenize equations of degree k to k, not maximal d?</b></summary> 

> No. I mean, yes, but then you will have different error vectors which need to be treated in a different way. So we probably need to deal with it one way or another.</details>
<br>
Now, this generalizes R1CS, and it also generalizes plonkish. The difference with R1CS is that equations do not have the form (linear)*(linear)=(linear), and difference with plonkish is that we do not need to spam the same equation over the each row, and are free to use them in a completely unstructured fashion.

As you can see, this arithmetization is pretty general, and supports basically anything. Let's talk a bit about cost model.

I suggest also the following **enhanced** arithmetization: together with the circuit, we get a collection of non-intersecting subsets $S_i$, to which (in the folding process) we will compute *partial witness commitments*. These can be used in multi-round challenge protocols like Origami or moon-moon.

<details><summary><b>A note for the curious reader.</b></summary> 

> We could actually declare every column in Sangria to be a different partial witness. It would trivialize loading it into final PlonK - as we would be free to choose independent commitment key for each subset, and we would promptly choose the same one. However, the cost of moving around a bunch of commitments instead of a single one likely outweight
</details>

## Cost model

There are two costs that are important in our exposition: one is of the folding itself, and other is of the final verifier. I believe the first cost largely dominates for the relevant applications (because we need to fold potentially millions of steps), and final verifier only needs to be *reasonably small*. Therefore, as long as we don't do something completely insane (like making full non-sparse R1CS-like circuit and trying to prove it in PlonK, incurring quadratic amount of suffering in process), it should be fine, and we should focus on the costs of folding. If you find yourself incurring quadratic amount of suffering frequently, I would suggest treating it by changing the final snark.

Let's check work that Novaish folding requires to do the fold a new step into already existing previous one. I will denote as $|w|$ witness size and $|E|$ the error vector size (i.e. amount of non-linear equations).

1. Compute witness commitment to a new step: $|w|$-sized MSM.
2. Compute cross-term commitment: $(d-1)$ MSMs of size $|E|$.
3. Do other stuff (negligible in comparison): i.e. hash some stuff, multiply few points by few constants and add them up.

I.e., for each new step we pay $1$ MSM of size $|w|$ and $(d-1)$ MSMs of size $|E|$.

Enhanced arithmetization (almost) does not incur additional costs for the prover, but it incurs additional costs to the verifier, because verifier basically only cares about (3).

Also, in this model I ignore the cost of actually computing the witness. It might be comparable in case we are using very dense gates (for example, have a quadratic gate of the form $\sum q_{ij} w_i w_j$ with all $q_{ij}$-s being nontrivial, or something even worse). This is an interesting, yet dubious endeavor, for it has a potential of incurring quadratic amount of suffering in the actual SNARK stage.

<details><summary><b>A note for the curious reader.</b></summary> 

> Also, this particular example can be easily reduced to R1CS, find out how! <br><br> But we can have non-trivial examples of this kind in higher degree (for example, determinant of a matrix can be computed in $n^3$ operations but only in $n^4$ if you are not allowed to divide).
</details>

Typically, the cost of computing witness should be negligible to an actual MSM.

## Cost model - advanced tricks

We should also understand, that not MSMs are equal with eachother - for example, MSM with coefficients being only $0$-s and $1$-s can be very fast to compute!

This effect is also felt in common proof systems like Groth16 or PlonK, though it is much smaller because typically we need to also construct factor polynomial $H(x)$, which has big coefficients even if original polynomials were small-coefficient ones.

Similar to this, we can skip a bit of work, but not all of it, in Novaish: namely, computing the witness commitment is much simpler. Computing cross-terms, on the other hand, is not simpler, and they, of course, tend to blow up quite fast.

### Supernova

But there are also specific cases where *a large subset of witness is actually 0*. Namely, imagine we are struggling with the same problem as Supernova - we have a circuit which is split into few parts, and only want to activate some particular parts of the circuit.

It can be done almost without overhead, using the following trick: create a selector variable $\lambda_i$, and use it instead of $t$ in the gates belonging to the $i$-th branch of computation.

Now, there will be an additional circuitry which picks which selector is $1$ (and all others are $0$), and we then can safely nullify the unneeded part of the computation.

Curious reader, of course, once again will ask about cross-terms: what if we are folding instances with different parts of the circuit activated? The answer is, in this case cross-terms can also be computed quite easily - in fact, they will all be exactly zero for the nullified part of the witness.

In fact, what I'm describing above is the Supernova construction, packed inside of a single Nova instance and feeling good about it.

### Better lookups

There is also a potential to make a new lookup argument, which is (sort of) independent of the size of the table being looked up. It will still likely be prover-linear in the final check unless we come up with some more tricks. I will describe it in a next part, when we take a look on Origami and moon-moon and possible directions of research there.

## Work done in this direction

I have started a complete rewrite of Nova to remove the dependency on bellperson and R1CS, but because I'm a terrible programmer it is not going that well. Any help in this direction will be appreciated.

# 2. Lookups

Let's review what we have now. In order to do lookups, we need a challenge oracle, and to have it, we have created partial witnesses. There are many things that we want to do with partial witnesses - for example, we can use them to load some step-dependent data in our circuit (using moon-moon's running hash), or generate challenges - either in moon-moon fashion, or step-by-step (which is Origami, though authors did not frame it in this way).

To understand the next part, let's do a small recollection of required tooling.

## Loading *step-independent* table in a circuit is free

Yes, it doesn't even require a partial witness! All we need to do is to check that the table in the final instance is $\text{TABLE}_{n} = t_n * \text{TABLE}$, where $t_n$ is our final relaxation coefficient.

Indeed, constant data is just multiplied by $t$ during the folding. Nice.

## Partial witnesses

Once again, what I'm describing is my understanding of the construction. It is not spelled out in the original text, but there are hardly any other ways to proceed.

Recall: the relaxed committed instance is a tuple $([w], [E], t, x)$. Triple $(w, E, t)$ will sometimes be called "extended witness", and $x$ is a public input. We modify this data, by keeping multiple $[w_i]$-s called partial witnesses (we could also have partial error vectors but I don't have any use for this currently). The definition of folding does not change, except now we need to fold all $[w_i]$'s (and of course they all participate in Fiat-Shamir for the random folding coefficient).

All other things remain in place.

<details><summary>🤔</summary> Maybe this ^ construction is what should be called moon-moon? It feels like a very diverse bag of tricks at this point, with partial witness at its crux. </details>

## Origami - a look on IVC

Anyways, Origami basically works like this: we have two subsets, "before the challenge" and "after the challenge". We have public inputs which are actually challenges, I will denote them $c$, but in actual construction $c[0] = \beta$, $c[1] = \gamma$.

The witnesses to the "before" and "after" subsets are called $w_0$ and $w_1$, and actual witness is the sum of these too.

We add an additional constraint that $c \overset{\text{hash}}{\longleftarrow} [w_0]$.

Technically, in Origami this hash also includes the whole transcript of the previous computation. I think it is not exactly needed (for the lookup that is localized in 1 step and thus can not be harmed by reactively changing the data in the previous steps), and thus will ignore it for now - it can be added if the need arises.

This construction, of course, can be clearly and cleanly added to the IVC: add an additional challenge input to the IVC circuit (and F step circuit), pass the data from it to the step circuit without changing.

Outer IVC circuit has an access to $[w_0]$ of the instance being folded in, and can sample a commitment and pass it into the public input of this instance, $u.x$.

<details><summary><big><big>Big question / proposal</big></big></summary>

We would like to actually use lookups *in the IVC circuit itself*, not only in the step circuit F - this would allow us to eliminate the curve cycle and shrink the final on-chain verifier from current millions of constraints to tens of thousands.

The question is, can we use the above construction to use lookups *in* IVC? Intuitively, there is no barrier - as this construction computes the challenge in the outer circuit, and then passes it to the inner one, we probably could create partial structure which is non-trivial on the IVC circuit itself.

It requires a security proof, I don't have one yet, but here is an intuition:

Imagine we have a satisfying instance-witness quadruple $(U_i, W_i, u_i, w_i)$, with $u_i.x$ containing (inside of a hash) the instance $U_i$ (and correct challenges generated from partial committed witnesses).

Then, because $w_i$ is satisfying, it is a trace of computation of the folding of $U_{i-1}$ and $u_{i-1}$ into $U_i$, and because challenges are now generated in plaintext by the final verifier, the long arithmetic is not broken in $w_i$.

So, it is a correct folding, now (warning: extractors do not actually work like this though we all wish they did, it is only an intuition!!!) if you have folded satisfying witness $U_i, W_i$ you could only produce it by having satisfying witnesses $u_{i-1}, w_{i-1}, U_{i-1}, W_{i-1}$. It remains to check that $u_{i-1}.x$ actually contains correct challenges, but this is clear, because it is guaranteed by execution trace $w_i$.

^ this definitely requires formalization, but my gut feeling says **should work** (though I sometimes wish for unrealistic things)

 </details>

 ## More efficient lookups!

 As promised, I provide a more efficient lookup argument. It is useful when the amount of lookups is small compared to the table size; though we incur the linear cost in the final check. My lack of proficiency with Caulk / Baloo prevents me from figuring out whether it is possible to improve the final check work too, but even with final prover needing to process linear in table size amount of data it is much better than doing it *in every single step*.

 So, here we go. I will use fractional version of permutation check, as it is more fit for my purposes. Let's recall how it works:

1. For each variable that is being looked up, log a value $a_i$, where $i$ ranges from $1$ to $k$, where $k$ is the amount of lookups.
2. Compute how many times each table entry $s_i$ has occured in a table $S$. Call this multiplicity $q_i$. Note: this $q_i$ sequence has linear size ($n$), but provided we didn't do a lot of lookups, most of the entries are actually zero.
3. Generate challenge $\gamma$.
4. Check that $\underset{i=1}{\overset{k}{\sum}} \frac{1}{\gamma - a_i} = \underset{i=1}{\overset{n}{\sum}} \frac{q_i}{\gamma - s_i}$.

I suggest that we abuse the fact that zero coefficients do not actually occure in the MSM. Let's denote the variables $p_i = \frac{q_i}{\gamma - s_i}$.

They are constrained by a following homogeneous quadratic equation: $p_i * (\gamma - s_i) = t * q_i$. Note, that if $p_i$ and $q_i$ are $0$ for the both instances being folded, then the cross-term for this equation is also, necessarily, zero.

Therefore, we can safely ignore all the terms in the sum for which there was no lookup - they do not do anything nor in MSM for the witness, nor they impact cross-terms.

The table itself also doesn't require any work except of precomputing its commitment $[(s_1, ..., s_n)]$ (so we need to do a linear amount of work once in the setup phase).

Sadly, we do not currently have a final snark having the same desirable property of ignoring $0$ coefficients (not sure about Hyperplonk, not aware enough to judge).

# Shifting blocks of data in memory: KZG in full force.

As you may have noticed, I am particularly fond of moving some array of data from one circuit to the other, without decommitting it and committing it back. There is, however, a particular restriction: the CRS for the piece of data you are trying to copy should be the same in both places. You can circumvent it - at a cost of the opening argument, so for KZG it is fine, and for IPA it is sad. In what follows, I will focus on KZG.

As a warmup, lets consider a following situation: our circuit is in plonkish, and we really want to use (ultra)plonk as a final verifier. However, we do not want to carry commitments to the separate columns along our whole folding process. The solution is simple - let's use the following CRS:

$[L_0(x)], [L_k(x)], [L_{2k}(x)], ...$ for the first column

$[L_1(x)], [L_{k+1}(x)], L_{2k+1}(x), ...$ for the second column

et cetera. Here, $L_i$ is a Lagrangian basis, as used in PlonK.

Now, in the end, separate our commitment into pieces corresponding to the columns, lets call them $[w] = [w_0] + [w_1] + ... [w_{k-1}]$ (and I hope it does not get confused with partial witnesses, as these only exist on the final step). Now, do a multipoint KZG-open, to check that $w_i(\mu^k x) = \tilde{w_{i}}(x)$, where $[\tilde{w_i}]$ is the actual polynomial commitment we are putting into $i$-th column, and PlonK of course will be used for the subgroup generated by $\mu^k$.

This is very useful by itself.

I'd like to recall my *final compression check* suggestion: when we have an IVC, we use *non-IVC Nova itself* as a final compression SNARK. Recall that moon-moon can emulate any unstructured circuit at ~3x (!needs clarification!) cost, overhead coming from necessity to load the permutation and to generate the challenge.

**Question**: is it true that if the original circuit has a lot of zeroes, this new circuit can be produced in a way that it also has a lot of zeroes? I.e. can we make "sparse-friendly" copy constraint argument?

Assume for a moment, that the answer to the following question is yes. Then, only one obstacle is still on the road - KZG Open is not sparse friendly - because if we have a vector with a lot of zeroes, the quotient polynomial is still not sparse.

However, shift can be made sparse friendly - instead of the Lagrangian shift, we can just use commitment basis $X_1, ..., X_n, \psi X_1, ..., \psi X_n, \psi^2 X_1, ..., \psi^2 X_n, ...$ (for some different toxic waste element $\psi$). Then, shifting by $\psi$ can be checked as pairing equation of the form $\langle A, \psi G \rangle = \langle B, G \rangle$, which is, in fact, sparse-friendly.

So, there is even an avenue to use really big lookups at a relatively small cost (though verifier necessarily becomes logarithmic in size, which is, probably, not desirable).