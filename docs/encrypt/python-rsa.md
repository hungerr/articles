## Python RSA加密

### RSAES-OAEP 和 RSAES-PKCS1-v1_5

`OAEP= Optimal Asymmetric Encryption Padding`

RSA的加密机制有两种方案一个是`RSAES-OAEP`，另一个`RSAES-PKCS1-v1_5`。`PKCS#1`推荐在新的应用中使用`RSAES- OAEP`，保留`RSAES-PKCS#1-v1_5`跟老的应用兼容。它们两的区别仅仅在于加密前编码的方式不同。而加密前的编码是为了提供了抵抗各种活动的敌对攻击的安全机制。

### RSASSA-PSS 和 RSASSA-PKCS1-v1_5

`PSS = Probabilistic Signature Scheme`

`PKCS#1`的签名机制也有种方案：`RSASSA-PSS`和`RSASSA-PKCS1-v1_5`。同样，推荐`RSASSA-PSS`用于新的应用,而`RSASSA-PKCS1-v1_5`只用于兼容老的应用。


### rsa

rsa包使用RSAES-PKCS1-v1_5加密

```PYTHON
import base64

import rsa


def encryptDemo(data):
    public_data = '''MIIBCgKCAQEA61BjmfXGEvWmegnBGSuS+rU9soUg2FnODva32D1AqhwdziwHINFa
D1MVlcrYG6XRKfkcxnaXGfFDWHLEvNBSEVCgJjtHAGZIm5GL/KA86KDp/CwDFMSw
luowcXwDwoyinmeOY9eKyh6aY72xJh7noLBBq1N0bWi1e2i+83txOCg4yV2oVXhB
o8pYEJ8LT3el6Smxol3C1oFMVdwPgc0vTl25XucMcG/ALE/KNY6pqC2AQ6R2ERlV
gPiUWOPatVkt7+Bs3h5Ramxh7XjBOXeulmCpGSynXNcpZ/06+vofGi/2MlpQZNhH
Ao8eayMp6FcvNucIpUndo1X8dKMv3Y26ZQIDAQAB'''
    new_data = "-----BEGIN PUBLIC KEY-----\n" + public_data + "\n-----END PUBLIC KEY-----"
    public_key = rsa.PublicKey.load_pkcs1_openssl_pem(new_data)
    encrpt = base64.b64encode(rsa.encrypt(data.encode("utf-8"), public_key)).decode("utf-8")
    return encrpt
```

### pycryptodomex

代替2.7的Crypto：
```
pip install pycryptodome
```

或者作为新包下载:
```
pip install pycryptodomex
```

生成RSA:
```
from Crypto.PublicKey import RSA

secret_code = "Unguessable"
key = RSA.generate(2048)
encrypted_key = key.export_key(passphrase=secret_code, pkcs=8,
                              protection="scryptAndAES128-CBC")

file_out = open("rsa_key.bin", "wb")
file_out.write(encrypted_key)
file_out.close()

print(key.publickey().export_key())
```

分离public key and private key：
```
from Crypto.PublicKey import RSA

key = RSA.generate(2048)
private_key = key.export_key()
file_out = open("private.pem", "wb")
file_out.write(private_key)
file_out.close()

public_key = key.publickey().export_key()
file_out = open("receiver.pem", "wb")
file_out.write(public_key)
file_out.close()
```

加密PKCS1_v1_5方式：
```python
import base64

from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5


def encryptDemo(data):
    public_data = '''MIIBCgKCAQEA61BjmfXGEvWmegnBGSuS+rU9soUg2FnODva32D1AqhwdziwHINFa
D1MVlcrYG6XRKfkcxnaXGfFDWHLEvNBSEVCgJjtHAGZIm5GL/KA86KDp/CwDFMSw
luowcXwDwoyinmeOY9eKyh6aY72xJh7noLBBq1N0bWi1e2i+83txOCg4yV2oVXhB
o8pYEJ8LT3el6Smxol3C1oFMVdwPgc0vTl25XucMcG/ALE/KNY6pqC2AQ6R2ERlV
gPiUWOPatVkt7+Bs3h5Ramxh7XjBOXeulmCpGSynXNcpZ/06+vofGi/2MlpQZNhH
Ao8eayMp6FcvNucIpUndo1X8dKMv3Y26ZQIDAQAB'''
    new_data = "-----BEGIN PUBLIC KEY-----\n" + public_data + "\n-----END PUBLIC KEY-----"
    public_key = RSA.importKey(public_data)
    cipher = PKCS1_v1_5.new(public_key)
    encrpt = base64.b64encode(cipher.encrypt(data.encode("utf-8"))).decode("utf-8")
    return encrpt
```

PKCS1_OAEP方式：
```PYTHON
import base64

from Cryptodome.Signature import pss
from Cryptodome.Hash import SHA256, SHA1
from Cryptodome.Cipher import PKCS1_OAEP
from Cryptodome.PublicKey import RSA


def encrtypt(data, public_code):
    public_key = RSA.importKey(public_code)
    cipher = PKCS1_OAEP.new(public_key, hashAlgo=SHA256, mgfunc=lambda x, y: pss.MGF1(x, y, SHA256))
    encrypt_data = base64.b64encode(cipher.encrypt(data.encode())).decode()
    return encrypt_data
```

RSA加密是有长度限定的，所以当data长度过长时，需要做分段加密:
```PYTHON
import base64

from Cryptodome.Signature import pss
from Cryptodome.Hash import SHA256
from Cryptodome.Cipher import PKCS1_OAEP
from Cryptodome.PublicKey import RSA


def encrtypt(data, public_code, length):
    public_key = RSA.importKey(public_code)
    encrypt_data = []
    cipher = PKCS1_OAEP.new(public_key, hashAlgo=SHA256, mgfunc=lambda x, y: pss.MGF1(x, y, SHA256))
    for i in range(0, len(str(data)), length):
        encrypt_data.append(base64.b64encode(cipher.encrypt(str(data)[i:i + length].encode())).decode())
    return encrypt_data
```

hashAlgo默认是`SHA1`, 注意兼容

### java Cipher兼容

Cipher类提供了加密和解密功能，利用Cipher类可完成des、des3、rsa和aes加密。通过获取Cipher类对象
```JAVA

private static byte[] encryptByPublic(byte[] data, PublicKey key) throws GeneralSecurityException{
    Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
    cipher.init(Cipher.ENCRYPT_MODE, key);
    return cipher.doFinal(data);
}
```
对象对应解释为：“算法/模式/填充模式” -- “DES/CBC/PKCS5Padding”

常见有以下参数：

* AES/CBC/NoPadding (128)
* AES/CBC/PKCS5Padding (128)
* AES/ECB/NoPadding (128)
* AES/ECB/PKCS5Padding (128)
* DES/CBC/NoPadding (56)
* DES/CBC/PKCS5Padding (56)
* DES/ECB/NoPadding (56)
* DES/ECB/PKCS5Padding (56)
* DESede/CBC/NoPadding (168)
* DESede/CBC/PKCS5Padding (168)
* DESede/ECB/NoPadding (168)
* DESede/ECB/PKCS5Padding (168)
* RSA/ECB/PKCS1Padding (1024, 2048)
* RSA/ECB/OAEPWithSHA-1AndMGF1Padding (1024, 2048)
* RSA/ECB/OAEPWithSHA-256AndMGF1Padding (1024, 2048)

```
(1)加密算法有：AES，DES，DESede(DES3)和RSA 四种
(2) 模式有CBC(有向量模式)和ECB(无向量模式)，向量模式可以简单理解为偏移量，使用CBC模式需要定义一个IvParameterSpec对象
(3) 填充模式:
        * NoPadding: 加密内容不足8位用0补足8位, Cipher类不提供补位功能，需自己实现代码给加密内容添加0, 如{65,65,65,0,0,0,0,0}
        * PKCS5Padding: 加密内容不足8位用余位数补足8位, 如{65,65,65,5,5,5,5,5}或{97,97,97,97,97,97,2,2}; 刚好8位补8位8
```

```
RSA/ECB/OEAPWithSHA-1AndMGF1Padding
```

在SUN JCE是这样解释的：`RSA/ECB/OEAPWithSHA-1AndMGF1Padding中的HASH=SHA1  MGF1=SHA1`

## Note

注意hashAlgo需一致

python PKCS1_OAEP的new方法：
```PYTHON
def new(key, hashAlgo=None, mgfunc=None, label=b'', randfunc=None):
    """Return a cipher object :class:`PKCS1OAEP_Cipher` that can be used to perform PKCS#1 OAEP encryption or decryption.

    :param key:
      The key object to use to encrypt or decrypt the message.
      Decryption is only possible with a private RSA key.
    :type key: RSA key object

    :param hashAlgo:
      The hash function to use. This can be a module under `Cryptodome.Hash`
      or an existing hash object created from any of such modules.
      If not specified, `Cryptodome.Hash.SHA1` is used.
    :type hashAlgo: hash object

    :param mgfunc:
      A mask generation function that accepts two parameters: a string to
      use as seed, and the lenth of the mask to generate, in bytes.
      If not specified, the standard MGF1 consistent with ``hashAlgo`` is used (a safe choice).
    :type mgfunc: callable

    :param label:
      A label to apply to this particular encryption. If not specified,
      an empty string is used. Specifying a label does not improve
      security.
    :type label: bytes/bytearray/memoryview

    :param randfunc:
      A function that returns random bytes.
      The default is `Random.get_random_bytes`.
    :type randfunc: callable
    """

    if randfunc is None:
        randfunc = Random.get_random_bytes
    return PKCS1OAEP_Cipher(key, hashAlgo, mgfunc, label, randfunc)
```

`hashAlgo`默认`SHA1`， `MGF1`默认和`hashAlgo`一致

```PYTHON
class PKCS1OAEP_Cipher:
    """Cipher object for PKCS#1 v1.5 OAEP.
    Do not create directly: use :func:`new` instead."""

    def __init__(self, key, hashAlgo, mgfunc, label, randfunc):
        """Initialize this PKCS#1 OAEP cipher object.

        :Parameters:
         key : an RSA key object
                If a private half is given, both encryption and decryption are possible.
                If a public half is given, only encryption is possible.
         hashAlgo : hash object
                The hash function to use. This can be a module under `Cryptodome.Hash`
                or an existing hash object created from any of such modules. If not specified,
                `Cryptodome.Hash.SHA1` is used.
         mgfunc : callable
                A mask generation function that accepts two parameters: a string to
                use as seed, and the lenth of the mask to generate, in bytes.
                If not specified, the standard MGF1 consistent with ``hashAlgo`` is used (a safe choice).
         label : bytes/bytearray/memoryview
                A label to apply to this particular encryption. If not specified,
                an empty string is used. Specifying a label does not improve
                security.
         randfunc : callable
                A function that returns random bytes.

        :attention: Modify the mask generation function only if you know what you are doing.
                    Sender and receiver must use the same one.
        """
        self._key = key

        if hashAlgo:
            self._hashObj = hashAlgo
        else:
            self._hashObj = Cryptodome.Hash.SHA1

        if mgfunc:
            self._mgf = mgfunc
        else:
            self._mgf = lambda x,y: MGF1(x,y,self._hashObj)

        self._label = _copy_bytes(None, None, label)
        self._randfunc = randfunc
```
