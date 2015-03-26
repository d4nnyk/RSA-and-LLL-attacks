# Lattice based attacks on RSA

This repo will host implementations and explanations of different RSA attacks using **lattice reduction** techniques (in particular **LLL**).

First, we'll see how **Coppersmith** found out that you could use lattice reduction techniques to attack a relaxed model of RSA (we know parts of the message, or we know parts of one of the prime, ...). And how **Howgrave-Graham** reformulated his attack.

Second we'll see how **Boneh and Durfee** used a coppersmith-like attack to factor the RSA modulus when the private key is too small (`d < N^0.929`). Followed by a simplification from **Herrman and May**.

If you want to use the implementations, see below for explanations on [Coppersmith](#coppersmith) and [Boneh-Durfee](#boneh-durfee). If you want to dig deeper you can also read my survey [here](rapport.pdf) (**warning: work in progress**).

I've also done some personal researches on the Boneh-Durfee algorithm and I do get better results. Check `research.sage`.

# Coppersmith

I've implemented the work of **Coppersmith** (to be correct the reformulation of his attack by **Howgrave-Graham**) in [coppersmith.sage](coppersmith.sage).

I've used it in two examples in the code:

## Stereotyped messages

For example if **you know the most significant bits of the message**. You can **find the rest of the message** with this method.

The usual RSA model is this one: you have a ciphertext `c` a modulus `N` and a public exponent `e`. Find `m` such that `m^e = c mod N`.

Now, this is the **relaxed model** we can solve: you have `c = (m + x)^e`, you know a part of the message, `m`, but you don't know `x`.
For example the message is always something like "*the password today is: [password]*".
Coppersmith says that if you are looking for `N^1/e` of the message it is then a `small root` and you should be able to find it pretty quickly.

let our polynomial be `f(x) = (m + x)^e - c` which has a root we want to find `modulo N`. Here's how to do it with my implementation:

```
dd = f.degree()
beta = 1
epsilon = beta / 7
mm = ceil(beta**2 / (dd * epsilon))
tt = floor(dd * mm * ((1/beta) - 1))
XX = ceil(N**((beta**2/dd) - epsilon))
roots = coppersmith_howgrave_univariate(f, N, beta, mm, tt, XX)
```

You can play with the values until it finds the root. The default values should be a good start. If you want to tweak:
* beta is always 1 in this case.
* `XX` is your upper bound on the root. **The bigger is the unknown, the bigger XX should be**. And the bigger it is... the more time it takes.

## Factoring with high bits known

Another case is factoring `N` knowing high bits of `q`.

The Factorization problem normally is: give `N = pq`, find `q`. In our **relaxed** model we know an approximation `q'` of `q`.

Here's how to do it with my implementation:

let `f(x) = x - q'` which has a root modulo q

```
beta = 0.5
dd = f.degree()
epsilon = beta / 7
mm = ceil(beta**2 / (dd * epsilon))
tt = floor(dd * mm * ((1/beta) - 1))
XX = ceil(N**((beta**2/dd) - epsilon)) + 1000000000000000000000000000000000
roots = coppersmith_howgrave_univariate(f, N, beta, mm, tt, XX)
```

What is important here if you want to find a solution:

*  we should have `q >= N^beta`
* as usual `XX` is the upper bound of the root, so the difference should be: |diff| < X

note: `diff = |q-q'|`

# Boneh Durfee

The implementation of **Boneh and Durfee** attack (simplified by **Herrmann and May**) can be found in [boneh_durfee.sage](boneh_durfee.sage). 

The attack allows us to break RSA and the private exponent `d`.
Here's why RSA works (where `e` is the public exponent, `phi` is euler's totient function, `N` is the public modulus): 
```
   ed = 1 mod phi(N)
=> ed = k phi(N) + 1 over Z
=> k phi(N) + 1 = 0 mod e
=> k (N + 1 - p - q) + 1 = 0 mod e
=> 2k [(N + 1)/2 + (-p -q)/2] + 1 = 0 mod e
```

The last equation gives us a bivariate polynomial `f(x,y) = 1 + x * (A + y)`. Finding the roots of this polynomial will allow us to easily compute the private exponent `d`.

The attack works if the private exponent `d` is too small compared to the modulus: `d < N^0.292`.

To use it:

1. look at the tests in [boneh_durfee.sage](boneh_durfee.sage) and make your own with your own values for the public exponent `e` and the public modulus `N`.
2. guess how small the private exponent `d` is and modify `delta` so you have `d < N^delta`
3. increase `m` (the bigger, the longer the program will need to run) until you get the bound on the determinant (a message will tell you if you don't. For reference: 

* for `delta = 0.26` you will need `m = 3`

* for `delta = 0.27` you will need `m = 6`

* for `delta = 0.275` you will need `m = 8`

* for `delta = 0.28` you will need `m = 12`

4. If you don't want to use values of `m` that high, you can also try to decrease `X` as it might be too high compared to the root of `x` you are trying to find. This is a last recourse tweak though.

5. If you have a private exponent `d` that might be even higher, do the tests for `delta = 0.28` for example. If it doesn't find anything, do an exhaustive search on higher values of `d`.

6. Once you have `x` and `y` you can easily insert them into the equation:

```
x [(N + 1)/2 + y] + 1 = e d
```

Here's an example on how you could use the function if you suspect your private exponent to be lower than `N^0.275`:

```
# problem with N and e known
P.<x,y> = PolynomialRing(Zmod(e))
A = int((N+1)/2)
pol = 1 + x * (A + y)

# tweak those values
delta = 0.275
m = 8
t = int((1-2*delta) * m)   

# last resort tweaks
X = 2*floor(N^delta)
Y = floor(N^(1/2))  

# find the solutions
solx, soly = boneh_durfee(pol, e, m, t, X, Y)
```

**PS**: You can also try to use `research.sage`. It might help you to use a lower `m` than necessary. 