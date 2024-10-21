# <img style="vertical-align:middle" src="/Assets/flaxa_zoom.png" width="30"  alt=""/> HTB - Weak RSA

- CTF: HackTheBox
- Challenge Name: Weak RSA
- Category: Cryptography
- Difficulty: Easy
## Synopsis

In this CTF challenge, we are given an RSA public key and an encrypted flag file. The goal is to decrypt the flag using Wiener's attack, which targets RSA keys with small private exponents. By exploiting this vulnerability, we can recover the private key and decrypt the encrypted message.

## Description

Can you decrypt the message and get the flag?
## Challenge

The challenge contains two files:

- `key.pub`: This file contains the public key used to encrypt the flag
- `flag.enc` This file contains the encrypted flag

The public key looks like this:

```
-----BEGIN PUBLIC KEY-----
MIIBHzANBgkqhkiG9w0BAQEFAAOCAQwAMIIBBwKBgQMwO3kPsUnaNAbUlaubn7ip
4pNEXjvUOxjvLwUhtybr6Ng4undLtSQPCPf7ygoUKh1KYeqXMpTmhKjRos3xioTy
23CZuOl3WIsLiRKSVYyqBc9d8rxjNMXuUIOiNO38ealcR4p44zfHI66INPuKmTG3
RQP/6p5hv1PYcWmErEeDewKBgGEXxgRIsTlFGrW2C2JXoSvakMCWD60eAH0W2PpD
qlqqOFD8JA5UFK0roQkOjhLWSVu8c6DLpWJQQlXHPqP702qIg/gx2o0bm4EzrCEJ
4gYo6Ax+U7q6TOWhQpiBHnC0ojE8kUoqMhfALpUaruTJ6zmj8IA1e1M6bMqVF8sr
lb/N
-----END PUBLIC KEY-----
```

The hexdump of the encrypted flag looks like this:

```
00000000  01 a2 5f ef 76 63 5b db  ea 7e e7 6b 5a c4 31 8a  |.._.vc[..~.kZ.1.|
00000010  07 c4 a8 d1 34 ce 49 a5  39 56 17 d1 d6 bf c6 5e  |....4.I.9V.....^|
00000020  15 47 f2 c2 15 e4 28 85  2b 33 4c 75 22 da 54 e9  |.G....(.+3Lu".T.|
00000030  02 08 24 be a0 c9 46 30  eb 56 50 a7 01 d6 be 6a  |..$...F0.VP....j|
00000040  40 ec b8 02 e1 f4 c0 c9  7c 6a 1a cf e4 99 d8 e7  |@.......|j......|
00000050  e5 85 7b c2 be ec 7f 2c  95 86 f0 f4 fc 59 45 a9  |..{....,.....YE.|
00000060  a9 8d 13 ef ef ac 58 38  08 78 e6 fc 2c ad ef 63  |......X8.x..,..c|
00000070  8f 4e 26 16 48 6c 32 b9  d3 8d c6 e5 5b 6f a4 1d  |.N&.Hl2.....[o..|
00000080  ee                                                |.|
00000081

```

## Solution

### Problem Setup

In this challenge, we are given an RSA public key (`key.pub`) with a modulus `n` and public exponent `e`, as well as an encrypted flag file (`flag.enc`). Our goal is to decrypt the flag using information derived from the public key.

### Step 1: Extract the RSA Parameters
First, we extract the modulus `n` and the public exponent `e` from the RSA public key file using `openssl`:

```bash
# Extract modulus and exponent
openssl rsa -pubin -in key.pub -text

```

The output is:

```
Public-Key: (1026 bit)
Modulus:
    03:30:3b:79:0f:b1:49:da:34:06:d4:95:ab:9b:9f:
    b8:a9:e2:93:44:5e:3b:d4:3b:18:ef:2f:05:21:b7:
    26:eb:e8:d8:38:ba:77:4b:b5:24:0f:08:f7:fb:ca:
    0a:14:2a:1d:4a:61:ea:97:32:94:e6:84:a8:d1:a2:
    cd:f1:8a:84:f2:db:70:99:b8:e9:77:58:8b:0b:89:
    12:92:55:8c:aa:05:cf:5d:f2:bc:63:34:c5:ee:50:
    83:a2:34:ed:fc:79:a9:5c:47:8a:78:e3:37:c7:23:
    ae:88:34:fb:8a:99:31:b7:45:03:ff:ea:9e:61:bf:
    53:d8:71:69:84:ac:47:83:7b
Exponent:
    61:17:c6:04:48:b1:39:45:1a:b5:b6:0b:62:57:a1:
    2b:da:90:c0:96:0f:ad:1e:00:7d:16:d8:fa:43:aa:
    5a:aa:38:50:fc:24:0e:54:14:ad:2b:a1:09:0e:8e:
    12:d6:49:5b:bc:73:a0:cb:a5:62:50:42:55:c7:3e:
    a3:fb:d3:6a:88:83:f8:31:da:8d:1b:9b:81:33:ac:
    21:09:e2:06:28:e8:0c:7e:53:ba:ba:4c:e5:a1:42:
    98:81:1e:70:b4:a2:31:3c:91:4a:2a:32:17:c0:2e:
    95:1a:ae:e4:c9:eb:39:a3:f0:80:35:7b:53:3a:6c:
    ca:95:17:cb:2b:95:bf:cd

```
### Step 2: Analyse the Parameters

Generally in RSA implementations the encryption exponent is set to a smaller value like 65537. Given that this encryption exponent is large it could indicate a non standard implementation and the decryption exponent could be small.

Wiener's attack specifically exploits RSA setups where the private exponent `d` is small relative to `n`. The attack is effective when:

```math
d < \frac{1}{3} n^{\frac{1}{4}}
```

### Step 3: Apply Wiener's Attack
Wiener's attack can be used to recover the private key `d` when the RSA public key has a small private exponent `d`. The `owiener` Python library implements Wiener's attack, which we can use to find `d`.

Here’s the code used for applying Wiener's attack:

```python
import owiener

# Given RSA parameters
n = 0x03303B790FB149DA3406D495AB9B9FB8A9E293445E3BD43B18EF2F0521B726EBE8D838BA774BB5240F08F7FBCA0A142A1D4A61EA973294E684A8D1A2CDF18A84F2DB7099B8E977588B0B891292558CAA05CF5DF2BC6334C5EE5083A234EDFC79A95C478A78E337C723AE8834FB8A9931B74503FFEA9E61BF53D8716984AC47837B
e = 0x6117C60448B139451AB5B60B6257A12BDA90C0960FAD1E007D16D8FA43AA5AAA3850FC240E5414AD2BA1090E8E12D6495BBC73A0CBA562504255C73EA3FBD36A8883F831DA8D1B9B8133AC2109E20628E80C7E53BABA4CE5A14298811E70B4A2313C914A2A3217C02E951AAEE4C9EB39A3F080357B533A6CCA9517CB2B95BFCD

# Attempt Wiener's attack
d = owiener.attack(e, n)

if d is None:
    print("Failed")
else:
    print("Hacked d={}".format(d))
```

The script attempts to recover the private exponent `d` using the `owiener.attack` function. If successful, we obtain the private key.

### Step 3: Decrypt the Flag
Once the private key `d` is obtained, we proceed to decrypt the flag. The flag is stored in the file `flag.enc`, and we first read its content as bytes. Then, we convert it to an integer `c` and decrypt it using the formula `m = c^d mod n`, where `m` is the plaintext message.

Here’s the code for decryption:

```python
# Read the encrypted flag
in_file = open("flag.enc", "rb")
data = in_file.read()
in_file.close()

# Convert the encrypted data to an integer
c = int.from_bytes(data, byteorder='big')

# Decrypt the message using the private key
decrypted_message = pow(c, d, n).to_bytes(200, 'big')
print(decrypted_message)
```

After decryption, the output is the plaintext flag. This process successfully recovers the original message using Wiener's attack, demonstrating the vulnerability of RSA with small private exponents.

### Summary
In this challenge, we exploited a vulnerability in RSA encryption using Wiener's attack to recover the private key and decrypt the flag. This highlights the importance of selecting a sufficiently large private exponent to prevent such attacks in RSA implementations.

### Output

```
Private key d found: 44217944188473654528518593968293401521897205851340809945591908757815783834933
b'\x01\xa2_\xefvc[\xdb\xea~\xe7kZ\xc41\x8a\x07\xc4\xa8\xd14\xceI\xa59V\x17\xd1\xd6\xbf\xc6^\x15G\xf2\xc2\x15\xe4(\x85+3Lu"\xdaT\xe9\x02\x08$\xbe\xa0\xc9F0\xebVP\xa7\x01\xd6\xbej@\xec\xb8\x02\xe1\xf4\xc0\xc9|j\x1a\xcf\xe4\x99\xd8\xe7\xe5\x85{\xc2\xbe\xec\x7f,\x95\x86\xf0\xf4\xfcYE\xa9\xa9\x8d\x13\xef\xef\xacX8\x08x\xe6\xfc,\xad\xefc\x8fN&\x16Hl2\xb9\xd3\x8d\xc6\xe5[o\xa4\x1d\xee'
b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02!\xcf\xb2\x98\x83\xb0o@\x9ag\x9aX\xa4\xe9{Dn(\xb2D\xbb\xcd\x06\x87\xd1x\xa8\xab\x87"\xbf\x86\xda\x06\xa6.\x04,\x89-)!\xb36W\x1e\x9f\xf7\xac\x9d\x89\xba\x90Q+\xacL\xfb\x8d~J9\x01\xbb\xcc\xf5\xdf\xac\x01\xb2{\xdd\xd3_\x1c\xa5SD\xa7YC\xdf\x9a\x18\xea\xdb4L\xf7\xcfU\xfa\x0b\xaap\x05\xbf\xe3/A\x00HTB{s1mpl3_Wi3n3rs_4tt4ck}'

```


## Solution Code

```python
import owiener

n = 0x03303B790FB149DA3406D495AB9B9FB8A9E293445E3BD43B18EF2F0521B726EBE8D838BA774BB5240F08F7FBCA0A142A1D4A61EA973294E684A8D1A2CDF18A84F2DB7099B8E977588B0B891292558CAA05CF5DF2BC6334C5EE5083A234EDFC79A95C478A78E337C723AE8834FB8A9931B74503FFEA9E61BF53D8716984AC47837B
e = 0x6117C60448B139451AB5B60B6257A12BDA90C0960FAD1E007D16D8FA43AA5AAA3850FC240E5414AD2BA1090E8E12D6495BBC73A0CBA562504255C73EA3FBD36A8883F831DA8D1B9B8133AC2109E20628E80C7E53BABA4CE5A14298811E70B4A2313C914A2A3217C02E951AAEE4C9EB39A3F080357B533A6CCA9517CB2B95BFCD

d = owiener.attack(e, n)

if d is None:
    print("Failed")
else:
    print("Hacked d={}".format(d))
    
    
in_file = open("flag.enc", "rb") 
data = in_file.read() 
in_file.close()
print(data)

# Convert the encrypted data to an integer
c = int.from_bytes(data, byteorder='big')

# Decrypt the message using the private key
decrypted_message = pow(c, d, n).to_bytes(200, 'big')
print(decrypted_message)
```

---
### [Home](/)