+++
title = "CTF Writeup / GoogleCTF 2018 / Perfect Secrecy"
description = "Done as a member of the duks team."
author = "Yolan"
date = 2018-06-24T20:30:00+02:00
tags = [
    "crypto",
	"ctf"
]
math = true
+++

After having great fun last year in Google CTF with a nice RSA challenge, and a couple of strange crypto schemes, and despite the lack of enthusiasm of my fellow team members, I decided to play again this year. And the first crypto challenge I solved was also about RSA, it said:

```bash
Perfect Secrecy
This crypto experiment will help you decrypt an RSA encrypted message.

nc perfect-secrecy.ctfcompetition.com 1337
```
And it provided us an [attachment](/ddl/ctf/googlectf/2018/perfect.zip), which contained a file called `flag.txt` (probably encrypted, since it contained seemingly random bytes), a `key_pub.pem` file, containing a RSA public key and a python file `challenge.py`. 

Let us first take a look at the key:
```bash
$> openssl rsa -pubin -text -modulus -noout < key_pub.pem
Public-Key: (1024 bit)
Modulus:
    00:da:53:a8:99:d5:57:30:91:af:6c:c9:c9:a9:fc:
    31:5f:76:40:2c:89:70:bb:b1:98:6b:fe:8e:29:ce:
    d1:2d:0a:df:61:b2:1d:6c:28:1c:cb:f2:ef:ed:79:
    aa:7d:d2:3a:27:76:b0:35:03:b1:af:35:4e:35:bf:
    58:c9:1d:b7:d7:c6:2f:6b:92:c9:18:c9:0b:68:85:
    9c:77:ca:e9:fd:b3:14:f8:24:90:a0:d6:b5:0c:5d:
    c8:5f:5c:92:a6:fd:f1:97:16:ac:84:51:ef:e8:bb:
    df:48:8a:e0:98:a7:c7:6a:dd:25:99:f2:ca:64:20:
    73:af:a2:0d:14:3a:f4:03:d1
Exponent: 65537 (0x10001)
Modulus=DA53A899D5573091AF6CC9C9A9FC315F76402C8970BBB1986BFE8E29CED12D0ADF61B21D6C281CCBF2EFED79AA7DD23A2776B03503B1AF354E35BF58C91DB7D7C62F6B92C918C90B68859C77CAE9FDB314F82490A0D6B50C5DC85F5C92A6FDF19716AC8451EFE8BBDF488AE098A7C76ADD2599F2CA642073AFA20D143AF403D1
``` 

So, we have a 1024 bit RSA key, which is a bit low, but not necessarily weak and still out of reach of a brute force factorization. It features a common exponent: 65537, so far nothing fishy.

We can now look into the `challenge.py` file. It has very few functions:
```python
def ReadPrivateKey(filename): ...
  
def RsaDecrypt(private_key, ciphertext): ...
  
def Challenge(private_key, reader, writer): ...
```

And a rather simple main function:
```python
def main():
  private_key = ReadPrivateKey(sys.argv[1])
  return Challenge(private_key, sys.stdin.buffer, sys.stdout.buffer)
``` 

The `ReadPrivateKey` function is using the [Cryptography.io](https://cryptography.io) library to read a private key file in a standard way, however the `RsaDecrypt` function is a bit more interesting:
```python
def RsaDecrypt(private_key, ciphertext):
  assert (len(ciphertext) <=
          (private_key.public_key().key_size // 8)), 'Ciphertext too large'
  return pow(
      int.from_bytes(ciphertext, 'big'),
      private_key.private_numbers().d,
      private_key.public_key().public_numbers().n)
```

As you can see, it is performing the RSA decryption function _manually_ by taking a ciphertext as bytes in big endian format, converting it into an integer $$c$$ and computing using `pow` the RSA decryption $$c^d \bmod{n}$$, for $$d$$ the private exponent and $$n$$ the RSA modulus.

So, this tells us two things:
 1. the ciphertext is in big endian.
 2. no padding was used and it is textbook RSA. 
 
As you all know textbook RSA without any padding has a lot of shortcomings. It is now time to dig into the bigger `Challenge` function called by the `main`: 

```python
def Challenge(private_key, reader, writer):
  try:
    m0 = reader.read(1)
    m1 = reader.read(1)
    ciphertext = reader.read(private_key.public_key().key_size // 8)
    dice = RsaDecrypt(private_key, ciphertext)
    for rounds in range(100):
      p = [m0, m1][dice & 1]
      k = random.randint(0, 2)
      c = (ord(p) + k) % 2
      writer.write(bytes((c,)))
    writer.flush()
    return 0

  except Exception as e:
    return 1
```

So, it first reads 2 bytes from the reader (which is `sys.stdin.buffer` in the `main`) and stores them as `m0` and `m1` respectively. It then proceeds to read the ciphertext from the same reader as being $$\frac{1024}{8}=128$$ bytes. That ciphertext gets decrypted into the `dice` variable and then for 100 rounds, it generates a random value $$k\in\{0,1,2\}$$, adds it to the Unicode code point of the byte selected by `[m0, m1][dice & 1]`, reduces it modulo $$2$$, and sends it back on the writer (which is `sys.stdout.buffer` in the `main`).

That's definitively fishy. So what's really going on there? 

We get 100 times the value $$(\operatorname{ord}(p) + k) \bmod{2}$$, with a different, random $$k\in\{0,1,2\}$$... But this actually leaks information about the value of $$\operatorname{ord}(p)$$, because the operation $$k \bmod{2}$$ does not result in a uniform distribution: indeed, we are expecting twice as many $$0$$s as $$1$$s, because when $$k$$ is equal to both $$0$$ and $$2$$, it is congruent to $$0 \bmod 2$$.   
So, if $$(\operatorname{ord}(p) + k) \bmod{2}$$ gives us more 0s than 1s, we can be pretty sure that $$\operatorname{ord}(p)\equiv 0 \bmod{2}$$. 
On the other hand, if $$\operatorname{ord}(p)\equiv 1 \bmod{2}$$, the addition with $$k$$ would shift the value and we would expect to get twice as many $$1$$s as $$0$$s on average.

So, we have an information leak depending on the number of $$0$$s and $$1$$s that this `Challenge` function writes to the writer. But what information do we actually get? We learn the parity of the value $$\operatorname{ord}(p)$$, but what's $$p$$ again? Well, it is set just above as being `p = [m0, m1][dice & 1]`, so it builds a list made of `m0` and `m1` (which are both inputs we provide in the first part of the `Challenge` function), and it sets $$p$$ as being the element at index `dice & 1`. And as mentioned above, `dice` is the integer resulting from the RSA decryption operation. So, `dice & 1` is equal to the bitwise AND of the plaintext and $$1$$, which is basically equal to $$0$$ if the **least significant bit** (LSB) of the plaintext is $$0$$ and is equal to $$1$$ if the LSB of the plaintext is equal to $$1$$ as well, since the AND truth table goes as follows: 

$$\begin{array}{c|c||c}
x & y & x\land y \\ \hline
0 &	0 &	0  \\
0 &	1 &	0  \\
1 &	0 &	0  \\
1 &	1 &	1  \\
\end{array}$$

Now, this means that we have in the end an oracle telling us what's the value of the LSB of the decrypted ciphertext!

And while we all know the famous padding oracle attacks against RSA PKCS and RSA OAEP, which are relying on the most significant bit of the padded plaintext to decrypt a ciphertext, there also exists an attack [exploiting the LSB](http://secgroup.dais.unive.it/wp-content/uploads/2012/11/Practical-Padding-Oracle-Attacks-on-RSA.html). This is done by iteratively narrowing down the range of possible plaintext down to just one. At each iteration, we cut down the remaining possibilities, and while we quickly converge to the corrects most significant bytes, we still have to perform all the 1024 iterations to be able to pinpoint just one plaintext.

So, we now have all the building blocks we need to build an attack against this service. 

Firstly, we have a decryption oracle that we can query.
<center>
![We can query the oracle with any ciphertext](/images/gctf18-perfectrsa/query.jpg)
</center>
The oracle will then decrypt the ciphertext and give us a lot of "somewhat" randomized answers, based on values we chose, e.g. 0 and 1.
<center>
![Processing by the oracle](/images/gctf18-perfectrsa/answers.jpg)
</center>
And we can deduce the right value out of the oracle's answers, allowing us to conduct the "LSB oracle" attack against RSA.
<center>
![We weight the different answers and guess the actual plaintext value](/images/gctf18-perfectrsa/interpret.jpg)
</center>

In the end, the following code, inspired from a [previous CTF](https://ctf.rip/sharif-ctf-2016-lsb-oracle-crypto-challenge/) and from the above [linked page explaining this LSB-oracle](http://secgroup.dais.unive.it/wp-content/uploads/2012/11/Practical-Padding-Oracle-Attacks-on-RSA.html), allows to query the netcat server, provide it with two consecutive bytes (we need to have a different parity betwen `m0` and `m1`, otherwise we don't leak any information) and interpret the oracle's answer in order to rebuild the actual plaintext out of its answers. 

And guess what? The flag was at the very end of the plaintext, so that it wouldn't leak before we had recovered the whole plaintext. ;)

All right, time to go to bed, now. I'll be following tomorrow with another write-up for the "DM collision" challenge.

```
b"\x00\x02a\xb4\x02\x0c3\xd2.:\xe7\x9eB'\xf2\xd5\x1c7`\xe4\r\xcd\xac\xd8\x7f\x02_L\x84q\xea\x8c\xb4\x1d}\x82\x90g\x170\x00.oO CTF{h3ll0__17_5_m3_1_w45_w0nd3r1n6_1f_4f73r_4ll_7h353_y34r5_y0u_d_l1k3_70_m337} Oo."
```

```python
import decimal
from socket import socket
import sys

keysize =1024

# from the flag.txt file read in big endian
c  = 119217309829194046348522363146747304937295954356236084387775000685800908617629318417110348204755477939424239274691807984378678076883364880141274514261178345172539063655348321636037215361924332171794520555913635132454086644353250127828076172493408546050292999559913199868787754769594591462488323013485213560406
# extracted with openssl rsa -pubin -text -modulus -noout < key_pub.pem
n = 0xDA53A899D5573091AF6CC9C9A9FC315F76402C8970BBB1986BFE8E29CED12D0ADF61B21D6C281CCBF2EFED79AA7DD23A2776B03503B1AF354E35BF58C91DB7D7C62F6B92C918C90B68859C77CAE9FDB314F82490A0D6B50C5DC85F5C92A6FDF19716AC8451EFE8BBDF488AE098A7C76ADD2599F2CA642073AFA20D143AF403D1
e = 65537

def oracle(c,i):
    # I'm opening a new socket each time
    r = socket()
    r.connect(('perfect-secrecy.ctfcompetition.com', 1337))
    # we first send our two bytes m0 and m1
    tosend = b'\x00'
    if r.send(tosend) == 0:
        print("borken socket")
        sys.exit()
    tosend = b'\x01'
    if r.send(tosend) == 0:
        print("borken socket")
        sys.exit()
    # we now send the ciphertext c
    tosend = c.to_bytes(keysize//8,'big')
    print("step", i,"sending:", tosend)
    if r.send(tosend) == 0:
        print("borken socket")
        sys.exit()
    # and now we listen to our oracle
    zero,one = 0,0 
    for rounds in range(100):
        a= r.recv(1)
        #print ("> ", a)
        if a == b'\x00':
            zero += 1
        if a == b'\x01':
            one+=1

    print("zero:",zero, "\n one:", one)
    if zero > one:
        return 0
    return 1

# Encrypt the integer 2 to exploint RSA's malleability
c_of_2 = pow(2,e,n)

def partial(c,n):
    k = n.bit_length() # that is 1024
    decimal.getcontext().prec = k # allows for 'precise enough' floats
    lower = decimal.Decimal(0)
    upper = decimal.Decimal(n)
    for i in range(k): # will take a while
        possible_plaintext = (lower + upper)/2
        if not oracle(c, i):
            upper = possible_plaintext # plaintext is in the lower half
        else:
            lower = possible_plaintext # plaintext is in the upper half
        c=(c*c_of_2) % n # multiply y by the encryption of 2 again, thanks to RSA's malleability
    # in the end we got the plaintext!
    return int(upper).to_bytes(128,'big')

# and we exploit it
print ("\nAnd we got:", partial((c*c_of_2)%n,n))
```


Drawings made with <span style="color:red">❤</span> by [NawDoodle](https://www.instagram.com/nawdoodle).