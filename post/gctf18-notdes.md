+++
title = "CTF Writeup / GoogleCTF 2018 / DM Collision"
description = "Done as a member of the duks team."
author = "Yolan"
date = 2018-06-30T23:30:00+02:00
tags = [
    "crypto",
	"ctf"
]
math = true
+++

The challenge said:
```bash
Can you find a collision in this compression function?

nc dm-col.ctfcompetition.com 1337
```
and gave us an [attachment](/ddl/ctf/googlectf/2018/dm_collision.zip) containing two python scripts: `not_des.py` and `challenge.py`. 

Firstly, let's have a quick peek into `not_des.py`: it seems to be a regular implementation of the DES cipher, but given its name, it means something has been tampered with... It's probably the S-Boxes, but we'll be able to come back to this later.

Now, the `challenge.py` file is a bit more interesting, since it contains the following functions:
```python
def ReadFlag(filename): ...

def Compress(reader): ...

def Challenge(flag, reader, writer): ...
```
and a relatively simple `main` function:
```python
def main():
  logging.basicConfig(stream=sys.stderr, level=logging.DEBUG)
  flag = ReadFlag(sys.argv[1])
  return Challenge(flag, sys.stdin.buffer, sys.stdout.buffer)
```
So, it seems that the server is simply passing the inputs to the `Challenge` function, which goes as follow:
```python
def Challenge(flag, reader, writer):
  try:
    b1 = Compress(reader)
    b2 = Compress(reader)
    b3 = Compress(reader)

    if b1.key + b1.input == b2.key + b2.input:
      writer.write(b'Input blocks should be different.')
      writer.flush()
      return 1

    if b1.output != b2.output:
      writer.write(b'No collision detected.')
      writer.flush()
      return 1

    if b3.output != [0] * BLOCK_SIZE:
      writer.write(b'0 pre-image not found.')
      writer.flush()
      return 1

    writer.write(flag)
    writer.flush()
    return 0
```
So, it seems the inputs are used in the Compress function to generate 3 different values `b1`, `b2` and `b3` satisfying a so-called `block` type, that contains a `key`, an `input` and an `output` value.

Then the first two blocks are compared in a first conditional statement to see whether they differ or not.

If they differ, their output are then compared to see whether they are the same or not, which means—as the error message says—that the blocks [collide](https://en.wikipedia.org/wiki/Collision_attack). 

And if they do, finally the latest block `b3` is checked against the all `0`s output and—as its error message says—this basically means that we need to [find the pre-image](https://en.wikipedia.org/wiki/Preimage_attack) of `0`.

So far so good, we need to dig into that `Compress` function now:
```python
def Compress(reader):
  """Davies-Meyer single-block-length compression function."""
  key = reader.read(KEY_SIZE)
  inp = reader.read(BLOCK_SIZE)
  output = Xor(DESEncrypt(inp, key), inp)
  return Block(key, inp, output)
```
That's quite straightforward: the `Compress` function is simply using the Davies-Meyer single-block-length compression function, and its [Wikipedia entry](https://en.wikipedia.org/wiki/One-way_compression_function#Davies–Meyer) even mentions the fact that the Davies-Meyer construction has the property that even when using a totally secure block cipher, one can compute fixed points for the construction, since for any value $$m$$, one can find a value of $$h$$ such that $$E_m(h) \oplus h = h$$, since we can simply set $$h = E_m^{-1}(0)$$. But remember that we have to find a collision and the pre-image of 0, and so this funny property might not necessarily help us there.

The next step at this point is obviously to dig into the `not_des.py` file to see how it differs from the usual [Data Encryption Standard](https://en.wikipedia.org/wiki/Data_Encryption_Standard). 
But let me first recall you quickly how DES operates. DES is a classical blockcipher and it is based on a 16 rounds [Feistel network](https://en.wikipedia.org/wiki/Feistel_cipher) and a so-called "cipher function". Without digging too much into the details, the Feistel network ensure us that the cipher behave as a strong pseudorandom permutation, while the cipher-function is doing the actual encryption work.

Here is a depiction of the DES encryption process with its Feistel network clearly visible, and for $$F$$ the cipher function:
<center>
![DES encryption process](/images/gctf18-notdes/des_encryption.jpg)
</center>
and when looking at the code inside of the `not_des.py` file, we can see it clearly, along with some comments to make it even clearer:
```python
def DESEncrypt(plaintext, key):
  if isinstance(key, bytes):
    key = Str2Bits(key)
    assert (len(key) == 64)
  if isinstance(plaintext, bytes):
    plaintext = Str2Bits(plaintext)

  # Initial permutation.
  plaintext = [plaintext[IP[i] - 1] for i in range(64)]
  L, R = plaintext[:32], plaintext[32:]

  # Feistel network.
  for ki in KeyScheduler(key):
    L, R = R, Xor(L, CipherFunction(ki, R))

  # Final permutation.
  ciphertext = Concat(R, L)
  ciphertext = [ciphertext[IP_INV[i] - 1] for i in range(64)]

  return Bits2Str(ciphertext)
```
So far, so good, this is plain DES encryption, without any difference.

So, it probably means that things are a bit different in the `CipherFunction`, right? So, DES `CipherFunction` works: it takes two inputs, a subkey of 48 bits and a block of 32 bits, it expands the block into 48 bits and it mixes the key with it by XORing them together. Then, the XORed value is passed through 8 [S-Boxes](https://en.wikipedia.org/wiki/Substitution_box) that will map their 6-bits input onto 4 output bits. So, we get again 32 bits, which are passed through a latest permutation step that rearranged them according to a fixed permutation called the P-box in order to ensure that each S-Box output are spread across 4 different S-boxes at the next round. We can depict it as follows:
<center>
![DES encryption process](/images/gctf18-notdes/des_cipher_function.jpg)
</center>
So, is anything deviating from that method in the code contained in our challenge? Well, not really:
```python
def CipherFunction(key, inp):
  """Single confusion-diffusion step."""
  assert (len(key) == 48)
  assert (len(inp) == 32)

  # Confusion step.
  res = Xor(Expand(inp), key)
  sbox_out = []
  for si in range(48 // 6):
    sbox_inp = res[6 * si:6 * si + 6]
    sbox = SBOXES[si]
    row = (int(sbox_inp[0]) << 1) + int(sbox_inp[-1])
    col = int(''.join([str(b) for b in sbox_inp[1:5]]), 2)

    bits = bin(sbox[row][col])[2:]
    bits = '0' * (4 - len(bits)) + bits
    sbox_out += [int(b) for b in bits]

  # Diffusion step.
  res = sbox_out
  res = [res[P[i] - 1] for i in range(32)]
  return res
```
as you can see, everything seems to be matching the DES cipher function.

But, if this is so, then it means that the S-boxes are not the same, since it is called "not DES" and that the code actually gives different output than DES, when run. Right?  
Not even! Each S-box perfectly matches the corresponding DES S-box, and so does the key schedule, the P-box, the expansion function, and the rest... And yet, it does not interoperate with DES, so we are probably missing something.

And I did! The S-boxes are taken from the `SBOXES` array, but that array is actually defined as being:
`SBOXES = [S6, S4, S1, S5, S3, S2, S8, S7]`

So, basically "Not DES" cipher function looks like this:
<center>
![Not DES encryption process](/images/gctf18-notdes/notdes_cipher_function.jpg)
</center>

So, now we have a DES with reordered S-Boxes and we need to find a Davies-Meyer collision and the pre-image of $$0$$ to get the flag.

The first one, the collision, isn't too hard: a DES key is actually made of 64 bits, of which only 56 are actually used, as mentioned (sic) in a comment in the `KeyScheduler` function:
```python
# Only 56 bits are used. A bit in each byte is reserved for pairity checks.
```
So, there are 8 remaining bits, which are usually either discarded or used as parity check bits. Here the comment mentions the parity check, but when looking for it, that parity check is nowhere to be found in the code, which means that we can very easily generate two different blocks that would bypass the first sanity check performed by the `Challenge` function.
This can be done for example by simply taking a given input and key, and building the second block with the same input and the almost same key, excepted for its last bit that we could flip.

Both block would have a different key `b1.key != b2.key`, but if the keys are differing only at a "parity check bit", the actual (not)DES encryption wouldn't differ between the two blocks and it would pass the test `b1.output == b2.output` as long as their `input` values are the same.

All right, now we can pass the first gate, but we still have to find a pre-image of $$0$$ in order to obtain our flag.

That might be a bit harder, given the Davies-Meyer construction, if we want to have $$E_m(h) \oplus h = 0$$, we actually need to have $$E_m(h)=h$$, which means that we need to find a fixed point of the notDES encryption.

Now, everybody learns in cryptography classes that DES has well-known "[weak keys](https://en.wikipedia.org/wiki/Weak_key#Weak_keys_in_DES)" such as `00..0` and `11..1`, for example. These weak keys are generally quoted because they happen to cause the DES encryption mode to act exactly as the DES decryption mode, which means that they are transforming the DES `CipherFunction` into an [involution](https://en.wikipedia.org/wiki/Involution_(mathematics)).

But another less known fact about the DES weak keys, is that they also cause DES to have at least $$2^{32}$$ fixed points per weak key. This was initially discussed during [Crypto'85](https://www.math.uzh.ch/doc/cd/crypto/HTML/C85.HTM) by Coppersmith and Rivest, after the latter found a short cycle using weak keys. That means that $$h_0 = h_j$$ for some $$j>0$$, where $$E_{k_1}(E_{k_0}(h_i))=h_{i+1}$$ for $$0 \leq i \leq j$$, and $$k_0$$,$$k_1$$ two complementary weak keys. 

This was then explained by Coppersmith in term of fixed points $$E_{k_0}(h)=h$$, and short cycles $$E_{k_1}(E_{k_0}(h))=h$$, occurring in the middle of the cycle generally when $$h = h_{\frac{j}{2}}$$. This means that we can rely on these Coppersmith cycles to find a fixed point!

So, we know it was possible in 1985 to find such cycles in DES. Furthermore since our notDES encryption scheme is using the **same key schedule** and cipher function as DES, it means that a DES weak key will also be a notDES weak key. And so, I tried to see if I could find such a cycle using the provided python implementation.

However, Python being Python, it was awfully slow! A million iteration would take ages and it soon became clear that this wouldn't lead to a reasonable computation time for a CTF.
In order to improve the performance, I ended up recoding notDES in Go, which wasn't too hard, as the Go implementation is very similar to the `not_des.py` file and tried to find some Coppersmith cycle by applying Rivest's experiment to notDES with the all $$0$$s and all $$1$$s weak key pair:
```C
func main() {
	c, _ := notdes.NewCipher([]byte{0, 0, 0, 0, 0, 0, 0, 0})
	c2, _ := notdes.NewCipher([]byte{255, 255, 255, 255, 255, 255, 255, 255})

	dst := make([]byte, 8)
	dst2 := make([]byte, 8)
	p := make([]byte, 8)
	src, _ := hex.DecodeString(os.Args[1])

	i := 0
	for i == 0 || !bytes.Equal(p, dst) && !bytes.Equal(p, dst2) && !bytes.Equal(dst2, dst) {
		i++
		copy(p, dst)
		c.Encrypt(dst, src[:])
		c2.Encrypt(dst2, dst[:])
		copy(src, dst2)
		if (i%100000 == 0){
			fmt.Printf("%d: %x from %x\n", i, dst, src)
		}
	}
	fmt.Println("We won! At iteration", i, "\nValues:", p, dst, dst2, src)
}
```
And sure enough, the Go implementation, once launched with 4 different arguments soon crunched DES encryption using 100% of my CPU, and with some 800M iterations per minutes, it didn't take too long before I got some positive results on input `12341234567445F8` after 10 minutes already:
```bash
We won! At iteration 1843856254 
Values: [156 50 9 2 110 10 24 166] [194 252 85 163 252 110 145 125] [194 252 85 163 252 110 145 125] [194 252 85 163 252 110 145 125]
 ```
So, as you can see, we've found a cycle where `dst == dst2`, which means that `dst` is a fixed point for the all $$1$$s weak key, and that means that we have found a preimage for the all $$0$$s block under the Davies-Meyer compression function! We have found $$\text{dst} \xrightarrow{\operatorname{E}_0} \text{dst}$$.

Another instance of my program kept running a bit longer and found another interesting case on input `DEDEDADADEDEDADA`:
```bash
We won! At iteration 3284689159
Values: [127 78 105 31 185 107 147 123] [0 107 237 194 162 101 166 10] [127 78 105 31 185 107 147 123] [127 78 105 31 185 107 147 123]
 ```
Which means that the encryption of `p` by the all $$0$$s key itself encrypted by the all $$1$$s key is equal to `p`, so we have a short cycle.
We have found $$\text{src} \xrightarrow{\operatorname{E}_0} \text{dst} \xrightarrow{\operatorname{E}_1} \text{src}$$.

This could also allow us to build a collision using two completely different keys, if the parity bit trick had failed to pass the two first conditional checks, since $$E_0(\text{dst}) = \text{src}$$ and $$ E_1(\text{dst}) = \text{src}$$ (this holds because we're using weak keys, and the encryption is equivalent to the decryption when doing so.)

In the end, we can simply get the flag with the following code:
```python
def win():
    r = socket()
    r.connect(("dm-col.ctfcompetition.com",1337))
    r.send(unhexlify(8*'00'))
    r.send(unhexlify(8*'00'))
    r.send(unhexlify(7*'00'+'01'))
    r.send(unhexlify(8*'00'))
    m = bytes([194, 252, 85 ,163 ,252 ,110 ,145 ,125])
    r.send(unhexlify(8*'ff'))
    r.send(m)
    a = r.recv(95)
    print('>',a)

#> b'CTF{7h3r35 4 f1r3 574r71n6 1n my h34r7 r34ch1n6 4 f3v3r p17ch 4nd 175 br1n61n6 m3 0u7 7h3 d4rk}'
```

And we can smile, because the flag is a reference to [Adele's "Rolling in the Deep"](https://www.youtube.com/watch?v=rYEDA3JcQqw) song.

Drawings made with <span style="color:red">❤</span> by NawDoodle.
