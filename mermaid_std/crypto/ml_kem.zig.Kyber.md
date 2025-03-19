嗯，我需要用Mermaid来画这个Zig函数的流程图。首先，我得仔细看看这个代码的结构。看起来这是一个Kyber算法的实现，包含公钥、私钥的生成，加密和解密的部分。

首先，函数Kyber接受一个参数p，返回一个结构体。结构体里有各种常量和类型定义，比如ciphertext_length、shared_length等。然后定义了PublicKey和SecretKey结构体，分别有encaps和decaps方法。KeyPair结构体处理密钥对的生成，包括deterministic和随机生成两种方式。

接下来是InnerPk和InnerSk，这两个是内部公钥和私钥的结构，包含加密和解密的具体实现。还有innerKeyFromSeed函数，用于从种子生成内部密钥对。

现在需要把这些部分转换成流程图。可能需要分几个部分来画：

1. 总体结构：Kyber结构体包含PublicKey、SecretKey、KeyPair等。
2. PublicKey的encaps流程：生成共享秘密，加密过程。
3. SecretKey的decaps流程：解密过程，验证密文。
4. KeyPair生成流程：确定性生成和随机生成。
5. 内部函数如InnerPk.encrypt和InnerSk.decrypt的具体步骤。

注意每个方法内部的步骤，比如encaps中处理seed，生成m，计算kr，加密得到ct，根据ml_kem标志决定如何生成shared_secret。decaps中解密得到m'，重新加密生成ct'，比较ct和ct'，最后生成共享密钥。

可能还需要考虑错误处理和条件分支，比如ml_kem是否为真时的不同处理。

现在需要将这些步骤用Mermaid的流程图语法表示出来。每个函数或方法作为一个节点，步骤之间的流程用箭头连接。条件判断用菱形节点，比如判断ml_kem是否为真。

可能需要分多个流程图，或者在一个大流程图中分块处理。但为了清晰，可能按模块分开画更合适。

例如，PublicKey.encaps的流程图：

开始 -> 生成m（根据seed是否存在） -> 计算kr -> 加密得到ct -> 根据ml_kem生成shared_secret -> 结束。

SecretKey.decaps的流程图：

开始 -> 解密得到m' -> 计算kr2 -> 生成ct2 -> 比较ct和ct2 -> 替换K''为z -> 生成shared_secret。

KeyPair生成的流程：

开始 -> 生成随机种子或使用确定性种子 -> 调用innerKeyFromSeed生成pk和sk -> 计算hpk -> 返回KeyPair。

内部函数innerKeyFromSeed的流程：

开始 -> 扩展种子 -> 生成rho和sigma -> 生成A矩阵 -> 生成s和e -> 计算t -> 设置pk和sk。

这样分步骤绘制每个主要函数的流程图，确保覆盖所有关键步骤和条件分支。
================================================
```mermaid
graph TD
    %% Kyber结构体定义
    Kyber --> PublicKey
    Kyber --> SecretKey
    Kyber --> KeyPair
    Kyber --> InnerPk
    Kyber --> InnerSk
    Kyber --> innerKeyFromSeed

    %% PublicKey流程
    subgraph PublicKey
        PK_encaps[encaps] --> CheckSeed{seed exists?}
        CheckSeed -->|Yes| CopySeed[Copy seed to m]
        CheckSeed -->|No| RandomM[Generate random m]
        PK_encaps --> ComputeKR[Compute (K', r) via SHA3-512]
        ComputeKR --> Encrypt[Encrypt m with r to get ct]
        Encrypt --> CheckMLKEM{ml_kem?}
        CheckMLKEM -->|Yes| ReturnKPrime[Return K' as shared_secret]
        CheckMLKEM -->|No| ComputeSS[Compute KDF(K' ‖ H(ct))]
        ReturnKPrime --> EncapsResult[Return EncapsulatedSecret]
        ComputeSS --> EncapsResult
    end

    %% SecretKey流程
    subgraph SecretKey
        SK_decaps[decaps] --> Decrypt[Decrypt ct to get m']
        Decrypt --> ComputeKR2[Compute (K'', r') via SHA3-512]
        ComputeKR2 --> EncryptAgain[Re-encrypt m' to get ct2]
        EncryptAgain --> CompareCT{ct == ct2?}
        CompareCT -->|No| ReplaceK[Replace K'' with z]
        CompareCT -->|Yes| KeepK[Keep K'']
        ReplaceK --> CheckMLKEM2{ml_kem?}
        KeepK --> CheckMLKEM2
        CheckMLKEM2 -->|Yes| ReturnKDoublePrime[Return K''/z]
        CheckMLKEM2 -->|No| ComputeSS2[Compute KDF(K''/z ‖ H(ct))]
        ReturnKDoublePrime --> DecapsResult[Return shared_secret]
        ComputeSS2 --> DecapsResult
    end

    %% KeyPair生成流程
    subgraph KeyPair
        Generate[generate/generateDeterministic] --> GetSeed[Get seed]
        GetSeed --> innerKeyFromSeed[Generate pk/sk from seed]
        innerKeyFromSeed --> ComputeHPK[Compute H(pk)]
        ComputeHPK --> ReturnKeyPair[Return SecretKey and PublicKey]
    end

    %% InnerPk加密流程
    subgraph InnerPk
        InnerPk_encrypt[encrypt] --> SampleNoise[Sample r, e1, e2]
        SampleNoise --> ComputeU[Compute u = Aᵀ r + e1]
        ComputeU --> ComputeV[Compute v = <t, r> + e2 + m]
        ComputeV --> Compress[Compress u and v]
        Compress --> ReturnCT[Return ciphertext]
    end

    %% InnerSk解密流程
    subgraph InnerSk
        InnerSk_decrypt[decrypt] --> DecompressCT[Decompress u and v]
        DecompressCT --> ComputeM[Compute m = v - <s, u>]
        ComputeM --> CompressM[Compress m]
        CompressM --> ReturnM[Return plaintext]
    end

    %% innerKeyFromSeed流程
    subgraph innerKeyFromSeed
        IKF_Start[Start] --> ExpandSeed[Expand seed via SHA3-512]
        ExpandSeed --> GenerateA[Generate matrix A from ρ]
        GenerateA --> SampleS[Sample secret vector s]
        SampleS --> ComputeT[Compute t = A s + e]
        ComputeT --> TransposeA[Transpose A]
        TransposeA --> SetKeys[Set pk and sk]
    end
```