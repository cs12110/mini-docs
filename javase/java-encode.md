# Java 加密

在开发上面经常遇到需要加密数据的场景,那么在这里记录一下常用的加密方式.

---

### 1. MD5

MD5 是单向加密的东西,即:加密之后,你能解密出来,算我输.

```java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Objects;

/**
 * md5转换器
 *
 * @author cs12110
 * @version V1.0
 * @since 2020-11-14 15:31
 */
public class Md5Util {

    private static MessageDigest digest = null;

    /**
     * 将数据转成md5字符串
     *
     * @param data 数据
     * @return String
     */
    public synchronized static String encode(String data) {
        if (Objects.isNull(data)) {
            return null;
        }
        if (Objects.isNull(digest)) {
            try {
                digest = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException nsae) {
                nsae.printStackTrace();
            }
        }
        digest.update(data.getBytes());
        return toHex(digest.digest());
    }

    /**
     * 转换成16进制数据
     *
     * @param hash arr
     * @return String
     */
    private static String toHex(byte[] hash) {
        StringBuilder buf = new StringBuilder(hash.length * 2);
        int i;

        for (i = 0; i < hash.length; i++) {
            if (((int) hash[i] & 0xff) < 0x10) {
                buf.append("0");
            }
            buf.append(Long.toString((int) hash[i] & 0xff, 16));
        }
        return buf.toString();
    }
}
```

输入样例

```java
2nd & 11
```

输出样例

```java
4d46f40ba67b36de19ae55b348a9b605
```

---

### 2. AES

AES: 一种双向加密的算法,我加密,你解密.

```java
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;

import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

/**
 * Aes加密/解密
 *
 * @author cs12110
 * @version V1.0
 * @since 2020-11-14 15:50
 */
public class AesUtil {

    private static final String KEY_ALGORITHM = "AES";

    private static final String KEY_RANDOM="SHA1PRNG";

    /**
     * AES 加密操作
     *
     * @param content 待加密内容
     * @param key     加密密钥
     * @return 返回Base64转码后的加密数据
     */
    public static String encode(String content, String key) {
        SecretKey secretKey = getSecretKey(key);
        try {
            Cipher cipher = Cipher.getInstance(KEY_ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            byte[] p = content.getBytes(StandardCharsets.UTF_8);
            byte[] result = cipher.doFinal(p);
            BASE64Encoder encoder = new BASE64Encoder();
            return encoder.encode(result);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * AES 解密操作
     *
     * @param content content
     * @param key     key
     * @return String
     */
    public static String decode(String content, String key) {
        SecretKey secretKey = getSecretKey(key);
        try {
            Cipher cipher = Cipher.getInstance(KEY_ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, secretKey);
            BASE64Decoder decoder = new BASE64Decoder();
            byte[] c = decoder.decodeBuffer(content);
            byte[] result = cipher.doFinal(c);

            return new String(result, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 生成加密秘钥
     *
     * @return SecretKey
     */
    private static SecretKey getSecretKey(final String key) {
        try {
            SecureRandom secureRandom = SecureRandom.getInstance(KEY_RANDOM);
            secureRandom.setSeed(key.getBytes());
            KeyGenerator generator = KeyGenerator.getInstance(KEY_ALGORITHM);
            generator.init(secureRandom);
            return generator.generateKey();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

输入样例

```java
public static void main(String[] args) {
  String key = "cs12110";
  String content = "橄榄球: 2nd & 11";

  String encode = AesUtil.encode(content, key);
  String decode = AesUtil.decode(encode, key);

  System.out.println("待加密字符串:" + content);
  System.out.println("加密后字符串:" + encode);
  System.out.println("解密后字符串:" + decode);
}
```

输出样例

```java
待加密字符串:橄榄球: 2nd & 11
加密后字符串:GUdQOBcmNV4NP+fH9dB23nmBoyuToA6j+Is8T3ih89A=
解密后字符串:橄榄球: 2nd & 11
```
