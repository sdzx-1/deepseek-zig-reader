好的，我现在要总结这个Zig语言编写的AES-NI代码的主要函数流程。首先，我需要大致浏览整个代码结构，理解各个模块和函数的作用。代码中定义了一个Block结构体，代表AES的块操作，然后有并行处理的BlockVec，以及密钥扩展相关的KeySchedule，接着是加密和解密的上下文结构体AesEncryptCtx和AesDecryptCtx，最后是AES128和AES256的具体实现。

首先，Block结构体：
- Block结构体内部使用两个u64的向量作为内部表示（Repr），每个块16字节。
- 提供了fromBytes和toBytes方法，用于字节和块之间的转换。
- 加密和解密方法（encrypt, encryptLast, decrypt, decryptLast），使用了内联汇编指令vaesenc、vaesenclast、vaesdec、vaesdeclast，这些是AES-NI指令集的指令。
- 还有基本的位操作，如xorBytes、xorBlocks、andBlocks、orBlocks，用于块的异或、与、或操作。
- parallel结构体处理并行加密解密，根据不同的CPU模型选择最优的并行块数，并提供了encryptParallel、decryptParallel等方法，处理多个块的并行操作。

接下来是BlockVec函数：
- 根据块数量生成一个向量类型，支持SIMD指令。
- 同样有fromBytes、toBytes、xorBytes等方法，处理多个块的转换和异或。
- 加密和解密方法在这里也是循环处理每个块，使用内联汇编指令。

然后是KeySchedule结构体：
- 负责密钥扩展，根据AES的轮数（10或14）生成轮密钥。
- expand128和expand256方法分别处理128位和256位密钥的扩展，使用vaeskeygenassist指令生成轮密钥。
- invert方法用于生成解密所需的逆轮密钥，通过vaesimc指令处理中间轮密钥。

AesEncryptCtx和AesDecryptCtx：
- 加密上下文初始化时生成密钥扩展，提供encrypt和encryptWide方法，处理单个或多个块的加密。
- encrypt方法进行轮密钥加、多轮加密，最后一轮使用encryptLast。
- encryptWide利用并行处理，对多个块同时加密，提高效率。
- 类似地，解密上下文使用逆轮密钥，进行解密操作。

AES128和AES256结构体：
- 定义了具体的密钥长度和轮数，提供initEnc和initDec方法来创建加密和解密上下文。

可能需要注意的地方：
- 代码中大量使用了Zig的内联汇编和向量操作，需要确保目标平台支持AES-NI指令集。
- 并行处理部分根据不同的CPU模型选择不同的并行块数，这可能影响性能优化。
- 密钥扩展过程中，对于不同密钥长度（128和256位）的处理不同，需要正确调用expand128或expand256。
- 解密上下文的初始化是通过反转加密的密钥调度实现的，使用了invert方法。

总结起来，代码的主要流程是：定义块操作，处理密钥扩展，实现加密和解密的上下文，利用AES-NI指令和并行处理优化性能。各个函数通过组合基本块操作和密钥轮数，完成AES的加密和解密过程。
================================================
这个Zig语言实现的AES-NI代码主要围绕AES块操作、并行处理、密钥扩展和加密/解密上下文展开。以下是核心函数流程的总结：

---

### **1. AES块操作（`Block` 结构体）**
- **功能**：表示单个AES块（16字节），提供基础操作。
- **关键方法**：
  - **`fromBytes`/`toBytes`**：字节序列与块表示的相互转换。
  - **加密/解密**：
    - `encrypt`/`encryptLast`：使用`vaesenc`和`vaesenclast`指令完成单轮和末轮加密。
    - `decrypt`/`decryptLast`：使用`vaesdec`和`vaesdeclast`指令完成单轮和末轮解密。
  - **位操作**：`xorBytes`、`xorBlocks`等实现块与字节序列或块之间的异或、与、或操作。
- **并行处理**：
  - `encryptWide`/`decryptWide`：对多个块使用同一轮密钥并行加密/解密。
  - 根据CPU模型（如Skylake、Icelake）动态选择最优并行块数（3-8块）。

---

### **2. 块向量（`BlockVec`）**
- **功能**：通过SIMD指令并行处理多个AES块。
- **关键方法**：
  - **转换**：`fromBytes`/`toBytes`支持多块与字节序列的转换。
  - **加密/解密**：循环调用`vaesenc`/`vaesenclast`等指令，逐块处理。
  - **位操作**：`xorBlocks`等对多块进行向量化位操作。

---

### **3. 密钥扩展（`KeySchedule`）**
- **功能**：生成加密/解密所需的轮密钥。
- **核心流程**：
  - **128位密钥**：`expand128`使用`vaeskeygenassist`指令生成11轮密钥。
  - **256位密钥**：`expand256`分两阶段生成15轮密钥。
  - **逆密钥生成**：`invert`通过`vaesimc`指令反转加密轮密钥，用于解密。

---

### **4. 加密/解密上下文（`AesEncryptCtx`/`AesDecryptCtx`）**
- **加密流程**：
  1. **初始化**：根据密钥生成轮密钥（`KeySchedule`）。
  2. **单块加密**：
     - 初始轮密钥加（`xorBlocks`）。
     - 执行多轮加密（`encrypt`），最后一轮用`encryptLast`。
  3. **多块加密**：利用`encryptWide`并行处理多个块。
- **解密流程**：
  1. **初始化**：从加密上下文反转轮密钥（`invert`）。
  2. **单块解密**：类似加密，但使用`decrypt`和`decryptLast`。
  3. **多块解密**：使用`decryptWide`并行处理。

---

### **5. AES-128/AES-256实现**
- **AES128/AES256结构体**：
  - 定义密钥长度（128/256位）和轮数（10/14轮）。
  - 提供`initEnc`/`initDec`创建加密/解密上下文。

---

### **性能优化**
- **SIMD指令**：依赖`vaesenc`、`vaesenclast`等AES-NI指令加速块操作。
- **并行处理**：根据CPU特性选择并行块数（如Icelake支持6块并行）。
- **内存布局**：使用`@Vector`类型和向量化操作减少循环开销。

---

### **总结**
代码通过分层设计（块→块向量→密钥→上下文）实现了高效的AES加密/解密，充分利用了AES-NI指令集和并行计算。核心流程为：密钥扩展生成轮密钥，逐轮应用加密/解密操作，并通过SIMD和动态并行策略优化性能。