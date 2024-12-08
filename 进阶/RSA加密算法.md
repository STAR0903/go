# 基础知识

### 非对称加密

RSA 属于非对称加密算法，主要特点是密钥成对使用：一个用于加密（公钥），另一个用于解密（私钥）。  使用公钥加密的数据只能用对应的私钥解密，反之亦然。

这一特性使 RSA 非常适合用于安全通信和数字签名。

### 密钥生成

RSA 密钥由公钥和私钥构成。公钥可以公开分发，而私钥必须严格保密。

Go 提供了 `crypto/rsa` 和 `crypto/rand` 包，允许我们生成和使用 RSA 密钥对。

# 生成 RSA 密钥对

生成一对 RSA 密钥，并将私钥和公钥分别保存到 `private.pem` 和 `public.pem` 文件中。

密钥长度建议为 2048 位或 4096 位，以确保安全性。

```
package learn_rsa

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"os"
)

const KeyBites = 4096

func generateRSAKeyPair() error {
	// 生成私钥
	privateKey, err := rsa.GenerateKey(rand.Reader, KeyBites)
	if err != nil {
		return err
	}
	// 生成公钥
	publicKey := privateKey.Public()

	// 私钥保存pem格式
	privateKeyFile, err := os.Create("private.pem")
	if err != nil {
		return err
	}
	defer privateKeyFile.Close()
	privateKeyBytes := x509.MarshalPKCS1PrivateKey(privateKey)
	if err != nil {
		return err
	}
	privateKeyPEM := &pem.Block{
		Type:  "RSA PRIVATE KEY",
		Bytes: privateKeyBytes,
	}
	pem.Encode(privateKeyFile, privateKeyPEM)

	// 公钥保存pem格式
	publicKeyFile, err := os.Create("public.pem")
	if err != nil {
		return err
	}
	defer publicKeyFile.Close()
	publicKeyBytes, err := x509.MarshalPKIXPublicKey(publicKey)
	if err != nil {
		return err
	}
	publicKeyPEM := &pem.Block{
		Type:  "RSA PUBLIC KEY",
		Bytes: publicKeyBytes,
	}
	pem.Encode(publicKeyFile, publicKeyPEM)

	return err
}
```

# 公钥加密数据

### EncryptOEAP 函数

**[EncryptOEAP](https://link.zhihu.com/?target=https%3A//pkg.go.dev/crypto/rsa%3Ftab%3Ddoc%23EncryptOAEP)** 函数用于加密一串随机的信息。

```
func EncryptOAEP(hash hash.Hash, random io.Reader, pub *PublicKey, msg []byte, label []byte) ([]byte, error)
```

我们需要为这个函数提供一些输入：

**hash** ：哈希函数，用了它之后要能保证即使输入做了微小的改变，输出哈希也会变化很大。SHA256 适合于此。

**random** ：生成随机数，确保这样相同的内容重复输入时就不会有相同的输出

**pub** ：之前生成的公钥

**msg**：需要加密的数据

**label** ：标签参数。标签参数可以包含不会加密的任意数据，但它为消息提供了重要的上下文。例如，如果给定的公钥用于加密两种类型的消息，则可以使用不同的标签值来确保攻击者不能将用于一种目的的密文用于另一种目的。解密时会同时验证密文和标签。如果不需要，可以为空。

### 代码

```
// filename 公钥所在文件
// data 需要加密的数据
func encryptWithPublicKey(filename string, msg []byte) ([]byte, error) {
	// 读取公钥
	publicKeyByte, err := os.ReadFile(filename)
	if err != nil {
		return nil, err
	}
	// 公钥转换到合适类型
	block, _ := pem.Decode(publicKeyByte)
	if block == nil || block.Type != "RSA PUBLIC KEY" {
		err = fmt.Errorf("Failed to decode PEM block containing public key")
		return nil, err
	}
	parsePublicKey, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	publicKey, ok := parsePublicKey.(*rsa.PublicKey)
	if !ok {
		err = fmt.Errorf("Could not assert public key as RSA")
		return nil, err
	}
	// 加密并返回数据和错误
	return rsa.EncryptOAEP(
		sha256.New(),
		rand.Reader,
		publicKey,
		msg,
		nil,
	)
}
```

### 加密数据长度限制

##### 限制因素

**公共模数的长度** ：公钥（`public key`）的模数 `n` 的长度（以字节为单位），它决定了 RSA 加密算法可以处理的最大消息大小。模数 `n` 的长度通常与密钥长度（例如 2048 位、4096 位等）相同。

**哈希函数的长度** ：在 OAEP 填充方案中，需要使用哈希函数。哈希长度通常是固定的，例如 SHA-256 的输出长度是 256 位（32 字节）。

**OAEP 填充的开销** ：OAEP 填充方案需要一些额外的字节用于生成填充信息，以确保加密后的密文无法被攻击者修改。

##### 计算公式

在使用 RSA-OAEP 填充时，消息的最大长度可以通过以下公式计算：

`最大消息长度 = 公共模数的长度（字节） - 2 * 哈希函数的输出长度（字节） - 2`

**公共模数的长度（字节）** ：即公钥的模数长度。例如，对于 2048 位的 RSA 密钥，模数长度为 256 字节。

**哈希函数的输出长度** ：假设使用 SHA-256，哈希长度是 32 字节（256 位）。

**减去常数二** ：是为了留出空间给 OAEP 填充过程中的其它信息，比如盐值（salt）和其他元数据。

##### 计算实例

假设我们使用 2048 位的 RSA 密钥（即公钥的模数长度是 2048 / 8 = 256 字节）并且使用 SHA-256 作为哈希函数（SHA-256 的输出长度为 32 字节）。那么最大消息长度就可以按上述公式计算：

```
最大消息长度 = 256 - 2 * 32 - 2
             = 256 - 64 - 2
             = 190 字节
```

因此，对于 2048 位的 RSA 密钥和 SHA-256 哈希函数，你可以加密的最大消息长度为 190 字节。

##### 为什么有这样的限制？

RSA-OAEP 填充方案增加了一些额外的安全性，确保即使密文被篡改或重放，攻击者也无法利用这些修改。在加密时，消息需要通过哈希和填充生成特定长度的数据，这样加密和解密过程都能够保证其安全性。而这个填充过程需要一定的字节空间，所以消息长度就受到了上述限制。

# 私钥解密数据

```
// 从 PEM 文件中加载私钥
func loadPrivateKey(fileName string) *rsa.PrivateKey {
	data, err := os.ReadFile(fileName)
	if err != nil {
		fmt.Println("Failed to read private key file:", err)
		return nil
	}

	block, _ := pem.Decode(data)
	if block == nil || block.Type != "RSA PRIVATE KEY" {
		fmt.Println("Failed to decode PEM block containing private key")
		return nil
	}

	// 解析私钥
	privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		fmt.Println("Failed to parse private key:", err)
		return nil
	}

	return privateKey
}

// 使用私钥解密
func decryptWithPrivateKey(ciphertext []byte, priv *rsa.PrivateKey) string {
	// 使用私钥解密数据，采用 OAEP padding
	plaintext, err := rsa.DecryptOAEP(
		sha256.New(),
		rand.Reader,
		priv,
		ciphertext,
		nil,
	)
	if err != nil {
		fmt.Println("Decryption failed:", err)
		return ""
	}

	return string(plaintext)
}
```
