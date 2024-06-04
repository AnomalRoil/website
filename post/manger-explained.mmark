+++
title = "Understanding and implementing Manger attack"
description = "Let's dive a bit into the details."
author = "Yolan"
date = 2018-04-05T15:00:00+02:00
tags = [
    "crypto"
]
aliases = [
    "/2018/04/05/manger-explained/"
]
math = true
+++


The RSA cryptosystem has had its fair share of attacks over the years, but among the most impressive, you can find the infamous Bleichenbacher attack [<a href="#biblio">Ble98</a>], which doomed PKCS v1.5 in 1998. Nineteen years later, <a href="https://robotattack.org/">the ROBOT attack</a> proved that the Bleichenbacher attack was still a concern today. Now, what alternatives to RSA PKCS v1.5 do we have? Well, its successor, RSA OAEP also known as RSA PKCS v2.1 is obviously a good candidate.

![](/images/manger.png)

RSA OAEP is an interesting scheme because it has been <a href="http://eprint.iacr.org/2000/061/">mathematically proven to be secure</a> against a chosen-ciphertext attack in the random oracle model.

Guess what? An attack against "weak implementations" of RSA OAEP also exists. This attack, while less well known than Bleichenbacher's because it never makes the headlines, is known as "Manger's attack" [<a href="#biblio">Man01</a>] after the name of its creator, James Manger, who <a href="http://archiv.infsec.ethz.ch/education/fs08/secsem/Manger01.pdf">published it in 2001</a>.

In this blog post, I will first explain each part of the attack and then guide you through their implementation in Go. We will end up with a fully functional attack just waiting for a so-called "oracle" to be plugged in. Typical oracles, in practice, are provided by timing information, but sometimes an error message eases the job.

First, let me first introduce the notion of oracle as I'll use it in this post:

> **Definition:** _**oracle**_  
> An _oracle_ is a black-box that will take an input and answer a specific, fixed question regarding its input, without being restricted in terms of knowledge. It can be used by any user indifferently, including any adversary.

In cryptography, the most common sorts of oracles used to attack different schemes are _padding oracles_ and _timing oracles_.

* The _padding oracles,_ as in Bleichenbacher's attack, take ciphertexts as input and reveal whether or not the padding is correct after their decryption.
* The _timing oracles_, as used by Kocher [<a href="#bilbio">Koc96</a>] for example, usually report the time needed to decrypt a specific ciphertext.


In the end, timing oracles can often be used as padding oracles, typically when the implementation returns an early error if the padding is wrong, leading an attacker to build a padding oracle out of the return timings of an implementation.

## Manger's Attack

Let $$ n$$ be an RSA modulus, with $$ e$$ and $$ d$$ its public and private exponents, respectively, and $$ k=\lceil\log_{256}n\rceil$$ be its byte length.
Then as long as the attacker has access to an oracle able to tell them whether $$ y \equiv c^d \mod n$$ is less than $$ B=2^{8(k-1)}$$ or not, then they are able to narrow down the range of possible messages corresponding to the ciphertext $$ c = m^e \mod n$$ to only a single message, $$ m$$, in $$ \mathcal{O}(\log(n))$$ queries.

Furthermore, the attack crucially relies on a well-known property of the RSA cryptosystem, namely its malleability [<a href="#bilbio">DDN91</a>]:

> **Definition: _malleability_**  
> For an encryption $$ E(m)$$ of the plaintext $$ m$$, the scheme $$ E$$ is _malleable_ if it is possible for a given function $$ f_1$$ to generate another ciphertext $$ f_1(E(m))$$ which yields the plaintext $$ f_2(m)$$, for a function $$ f_2$$ without requiring knowledge of $$ m$$ at any point.
> The property of being malleable is called _malleability_.

This is the case for the RSA cryptosystem, since we operate in the quotient ring $$ \mathbb{Z}/n\mathbb{Z}$$. For a given public key $$ (e,n)$$, the plaintext $$ m$$ is encrypted as $$ E(m)\equiv m^e \mod n$$. We can construct the ciphertext $$ m\cdot x$$ for any $$ x$$, as being $$ E(m) \cdot x^e \mod n = (mx)^e \mod n \equiv E(mx)$$.

Now let me formally describe the attack:
Initially we only know one ciphertext, $$ c$$, we would like to decrypt without access to the private key, but with access to an oracle able to tell us, when we send it a value $$ a$$, whether the decrypted value $$ b \equiv a^d \mod n$$ satisfies $$ b \geq B$$ or not.
I'll now describe the attack as presented in [<a href="#bilbio">Man01</a>] when $$ 2B<n$$, which is usually the case since the size of most RSA moduli is an exact multiples of $$ 8$$ bits, rendering $$ n$$ at least $$ 128$$ times bigger than $$ B$$.
If this is not the case, then the attack is still possible but has to deal with multiple intervals, cf. [<a href="#bilbio">Man01</a>] section 3.2.
So, when that assumption holds, the attack is performed in three steps as follows:

### Step 1

We firstly have to try to multiply the unknown plaintext $$ m$$ with
$$ f_{1,1}= 2,\; f_{1,2} = 2^2,\; \ldots,\; f_{1,i}= 2^i, \; \ldots$$
by sending the values $$ f_{1,i}^e\cdot c \mod n$$ to the oracle for $$ i = 1, 2, \ldots$$ until it returns that $$ f_{1,i}\cdot m \geq B \mod n$$. This is true as soon as $$ f_{1,i}^e\cdot c \geq B \mod n$$ thanks to the malleability of the RSA cryptosystem.
This ensures us that $$ f_{1,i}\cdot m \in \left[B,2B\right[$$, so we know that $$ \frac{f_{1,i}}{2}\cdot m \in \left[\frac{B}{2}, B\right[$$.

This can easily be done in Go as follows:

```go
f1 := new(big.Int).SetInt64(int64(2))
for !tryOracle(f1, c, e, N, ourOracle) {
  f1.Mul(two, f1)
}
```

Where the <code>tryOracle(f, c, e, N, ourOracle)</code> function is sending the ciphertext $$ f^e\cdot c \mod n$$ to our oracle which returns, for $$ c = m^e \mod n$$ whether $$ f\cdot m \geq B \mod n$$ or not.

### Step 2

We now try to multiply the unknown plaintext $$ m$$ with
$$ f_{2,j} = \left(\left\lfloor\frac{n+B}{B}\right\rfloor + j \right)\cdot \frac{f_{1,i}}{2}, \quad\text{ for }j \in \mathbb{N}^*$$
by exploiting the malleability of the RSA cryptosystem.
To do so, we send $$ f_{2,j}^e \cdot c \mod n$$ to the oracle, for $$ j = 1, 2, \ldots$$, until it returns that $$ f_{2,j}\cdot m < B$$.
It then necessarily implies that the modulo reduction "wrapped" $$ f_{2,j}\cdot m$$ into something smaller than $$ B$$, since by definition of $$ f_{1,i}$$ and $$ f_{2,j}$$, we know that $$ f_{2,j} \cdot m \geq B$$.
In turn, it then implies that $$ f_{2,j} \cdot m \in \left[n, n+B\right[$$.

Note that this step necessarily terminates, taking at most $$ \left\lceil\frac{n}{B}\right\rceil$$ oracle queries, since $$ n$$ will always be exceeded when $$ f_{2,j} = \left\lceil\frac{2n}{B}\right\rceil \cdot \frac{f_{1,i}}{2}$$.

This is also easily implemented in Go:

```go
nB := new(big.Int).Add(N, B)
nBB := new(big.Int).Div(nB, B)
f2 := new(big.Int).Mul(nBB, f12)

for tryOracle(f2, c, e, N) {
  f2.Add(f2, f12)
}
```

### Step 3

We finally want to narrow down the range of possible messages to just one.
This is done iteratively, approximately dividing the range by two at each step. It uses a heuristic approach to define its parameters and has not been formally proven by Manger in his article.
This implies defining $$ m_{min, 1} = \left\lceil\frac{n}{f_{2, j}}\right\rceil$$ and $$ m_{max, 1} = \left\lfloor\frac{n+B}{f_{2, j}}\right\rfloor$$ and then as long as the range $$ \left[m_{min,k}, m_{max,k}\right]$$ is containing more than one value, one can do the following:

1. Choose a temporary multiple $$ f_{tmp, k}=\left\lfloor\frac{2B}{m_{max, k}-m_{min, k}}\right\rfloor$$;
2. Compute a boundary point $$ i_k = \left\lfloor \frac{f_{tmp, k}\cdot m_{min, k}}{n} \right\rfloor$$;
3. Compute a value $$ f_{3, k}$$, which is so that $$ f_{3,k}\cdot m$$ is spanning a single boundary point at $$ i_kn+B$$, as follows: $$ f_{3, k}=\left\lceil \frac{i_kn}{m_{min, k}} \right\rceil$$;
4. Send the value $$ f_{3, k}^e\cdot c$$ to the oracle, if it returns $$ f_{3, k}\cdot m \geq B$$, then one can set $$ m_{min, k+1} = \left\lceil \frac{i_kn+B}{f_{3, k}} \right\rceil$$ and go back to 1. Else, if it returns $$ f_{3, k}\cdot m < B$$, once can set $$ m_{max, k+1} = \left\lfloor \frac{i_kn+B}{f_{3, k}} \right\rfloor$$ and go back to 1.


This is the longest part of the computation and can be implemented as follows in Go (using notations as  close to Manger's article as possible):

```go
mmin := divCeil(N, f2)
mmax := new(big.Int).Div(nB, f2)
BB := new(big.Int).Mul(two, B)
diff := new(big.Int).Sub(mmax, mmin)

for diff.Sub(mmax, mmin).Cmp(zero) > 0 {
 ftmp := new(big.Int).Div(BB, diff)
 ftmpmmin := new(big.Int).Mul(ftmp, mmin)
 i := new(big.Int).Div(ftmpmmin, N)
 iN := new(big.Int).Mul(i, N)
 f3 := divCeil(iN, mmin)
 iNB := new(big.Int).Add(iN, B)
 if tryOracle(f3, c, e, N) {
  mmin = divCeil(iNB, f3)
 } else {
  mmax.Div(iNB, f3)
 }
}
```

When this iterating process terminates, the range of possible messages $$ \left[m_{min,k}, m_{max,k}\right]$$ only spans one value, $$ m$$ the secret message. On average, the attack requires approximately $$ \ell$$ queries, for $$ \ell$$ the bit-size of the RSA modulus.

## The code

My complete, working, heavily commented implementation of <a href="https://github.com/kudelskisecurity/go-manger-attack">Manger's attack in Go can be found on Github</a> and is compatible with generic oracles!

Note that in this implementation, I provide a test file `attack_test.go` where I am crafting the oracle to tell you whether the most significant byte of the decrypted, padded data is zero or not. This is equivalent to testing whether the decrypted value $$ b \equiv a^d \mod n$$ satisfies $$ b \geq B$$ or not, by definition of $$ B$$ and $$ k$$. This allows me to attack a modified version of Go's crypto library (to be found in the package <code>moddedrsa</code>). This file is a black-box test and it is probably a good example (excepted for the <a href="https://github.com/golang/go/wiki/CodeReviewComments#import-dot">infamous dot import</a>) if you want to implement the attack using your own oracle.

In the end, in order to use it with your very own oracle, you just need to implement the "Oracle" <a href="https://tour.golang.org/methods/9">interface</a> in your own Go program and call the attack using your oracle.

<a href="https://twitter.com/anomalroil">Let me know</a> if you used this code, I'm curious about the oracles you are using!

<h2 id="biblio">Bibliography</h2>

<table style="width: 100%;">
<tbody>
<tr>
<td style="width: 75pt;">[Ble98]</td>
<td>Daniel Bleichenbacher. “<a href="https://link.springer.com/content/pdf/10.1007/BFb0055716.pdf">Chosen Ciphertext Attacks Against Protocols Based on the RSA Encryption Standard PKCS #1</a>”.
In: CRYPTO 1998. Ed. by Hugo Krawczyk.
Vol. 1462. LNCS. Springer, Heidelberg, Aug. 1998.</td>
</tr>
<tr>
<td>[Man01]</td>
<td>James Manger. “<a href="https://link.springer.com/content/pdf/10.1007/3-540-44647-8_14.pdf">A Chosen Ciphertext Attack on RSA Optimal Asymmetric Encryption Padding (OAEP) as Standardized in PKCS #1 v2.0</a>”.
In: CRYPTO 2001. Ed. by Joe Kilian.
Vol. 2139. LNCS. Springer, Heidelberg, Aug. 2001.</td>
</tr>
<tr>
<td>[Koc96]</td>
<td>Paul C. Kocher. “<a href="https://link.springer.com/content/pdf/10.1007/3-540-68697-5_9.pdf">Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems</a>”.
In: CRYPTO 1996. Ed. by Neal Koblitz.
Vol. 1109. LNCS. Springer, Heidelberg, Aug. 1996.</td>
</tr>
<tr>
<td>[DDN91]</td>
<td>Danny Dolev, Cynthia Dwork, and Moni Naor. “<a href="https://www.researchgate.net/profile/Danny_Dolev/publication/2332888_Non-Malleable_Cryptography/links/54981cd60cf2eeefc30f712e/Non-Malleable-Cryptography.pdf">Non-Malleable Cryptography (Extended Abstract)</a>”.
In: 23rd ACM STOC.
ACM Press, May 1991.</td>
</tr>
</tbody>
</table>
