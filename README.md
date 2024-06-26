# awesome-device

内存映射 I/O（mmio）是 CPU 访问外设的一种典型方式。它的主要特征是将一块物理地址空间重定向到外设地址空间，使 CPU 能够透明地访问外设。这样的硬件设计对于系统软件的编程产生了一种正面的影响：**访问外设现在可以和访问其他一些特殊的内存区域统一起来了**。这种软硬协同的方式很类似带 MMU 的 CPU 对物理内存中页表的理解：读写这块地址空间就会修改硬件的工作状态。使用它的方式又像是在运行时检查那些由链接器放在指定位置的信息。

举例来说，`qemu-system-riscv64` 的 `virt` 仿真器，0x1000_0000 开始的一段地址空间映射到一个 8 位 16550 兼容串口外设，这也是这台虚拟机的主要控制台外设。对于系统软件开发者来说，这可以理解为 0x1000_0000 处放置着一个由手册描述的硬件寄存器决定的数据结构，读写这个数据结构硬件就能操作外设。同时，不同的物理硬件上，同一种外设的结构是相同的，只是映射的物理地址空间可能不同。

那么——一个很自然的想法——为什么不直接把这个结构定义出来做成库，以便在不同的软件和硬件上复用呢？

过去，已经有很多开发者开发了许多 mmio 驱动软件。这些驱动软件总是试图通过一个变量来描述在地址空间上的位置。这个看似正常的变量其实带来不少问题，比如在那些不具备跨平台需求，追求极致性能和尺寸的嵌入式程序中。大量外设其实位于已知的、固定的位置，它们的地址完全可以作为一个常量随处内联，那些通过传入基址构造的驱动就无法内联。另外，对于一个硬件上多个相对位置固定的外设，也没有一种简单的方式将它们的驱动组合起来。典型的是 DMA 控制器，它总是需要和各种外设联动，开发 DMA 的驱动总是非常麻烦而且很难做到优雅。

在我看来，驱动作者应该少管闲事，而不是猜测用户要怎么使用外设。驱动要做的就是由硬件决定了的部分：寄存器的功能定义，以及它们的协同关系。驱动应该仅根据外设的手册来提供一个结构体以完成外设的操作，将结构体映射到内存空间的方式则由系统软件开发者控制。极端条件下，一个包括所有外设的链接脚本可以将整个定位过程提前到运行之前。组合多个驱动也变得简洁而且零开销：只需要把这些结构体放到一个大结构体里，并且填充合适的间距就可以了。驱动可以以递归的结构任意组合，并把任何操作下降到寄存器。

这个库就是按这个想法实现的驱动的列表。其中一些库已经发布，并在具体的项目中发挥作用：

- aclint: [![Latest version](https://img.shields.io/crates/v/aclint.svg)](https://crates.io/crates/aclint)
- plic: [![Latest version](https://img.shields.io/crates/v/plic.svg)](https://crates.io/crates/plic)
- sifive-test-device: [![Latest version](https://img.shields.io/crates/v/sifive-test-device.svg)](https://crates.io/crates/sifive-test-device)
- uart16550: [![Latest version](https://img.shields.io/crates/v/uart16550.svg)](https://crates.io/crates/uart16550)

---

## 关于自动化外设驱动开发的设想

在当今快速发展的嵌入式系统领域，外设驱动开发已成为一项既无趣又充满挑战的任务。这项工作不仅要求开发者具备深厚的硬件知识和对底层细节的精确把握，而且调试过程往往复杂且耗时。随着 RISC-V 架构的嵌入式设备——尤其是国产设备——的崛起，外设的多样性正在以前所未有的速度增长。这一变化使得传统的外设驱动开发方法显得愈发低效，尤其是在跨硬件平台复用驱动代码方面存在显著的局限性。

传统的外设驱动开发通常涉及对硬件手册的深入解读，以及大量的手动编码工作，以实现对特定外设的控制。这种方式在面对硬件更新换代时，往往需要重复劳动，且易于出错。随着设备种类的增加，这种劳动密集型的开发模式不仅效率低下，而且难以保证代码的一致性和可维护性。

为了解决这些问题，我们提出了一种自动化外设驱动开发的新设想。该设想基于硬件规范文档和硬件描述文件，利用自动化工具链来自动生成适应不同硬件平台的驱动代码。通过这种方法，我们可以将开发者从繁琐的硬件适配工作中解放出来，让他们能够专注于更高级别的系统功能开发和优化。

自动化工具将首先解析硬件规范文档中的寄存器布局和功能描述，生成标准化的数据结构和操作接口，同时将对应的描述文字拷贝到字段和方法的文档注释，使得自动生成的代码人类可读。这些数据结构将直接映射到硬件的寄存器地址空间，系统软件开发者可以通过这些结构体来操作外设，无需人工完成解读规范文档的工作，容易保证底层代码的正确性和规范性。自动化工具不仅止步于生成可读的驱动代码。为了确保生成的驱动代码的正确性和可靠性，该工具还应自动生成单元测试代码。这些单元测试代码将基于硬件规范文档中的操作描述，自动执行一系列测试用例，验证驱动与硬件的交互是否符合预期。通过这种方式，开发者可以在早期阶段发现并修复潜在的问题，提高驱动的稳定性和质量。最后，自动化工具还将生成项目配置文件，方便开发者将代码发布到库分发平台。这样，其他开发者可以直接从平台获取并使用这些经过验证的驱动代码，无需从头开始编写。这不仅加速了开发流程，还有助于构建一个健康活跃的开发者社区，鼓励知识共享和协作。

最后，为了实现外设驱动的自动化集成，一个自动配置工具将读取硬件描述文件（如设备树文件）自动从库分发平台或本地路径获取相应的外设驱动代码，并根据硬件描述文件中的信息组合这些驱动，生成开发板级的外设映射。这一过程将自动完成，确保所有外设在开发板上的正确定位和配置，极大地提高了硬件初始化的准确性和开发效率。通过这一系列的自动化步骤，我们的目标是为嵌入式系统开发者提供一个全面、高效、可靠的外设驱动开发解决方案，从而推动 RISC-V 及其他架构嵌入式设备的发展和创新。

## 自动化工具对一致、可解析的规范文档的需求

为了实现这样的自动化外设驱动开发工具，一个关键的前提是拥有统一的、清晰的硬件规范文档。这些文档不仅是开发者理解和操作硬件的基础，也是自动化工具能够准确解读和转换信息的关键。因此，硬件规范文档应当采用一种结构化的标签语言来编写，如 XML、JSON 或 TOML，这些格式具有良好的可读性和可解析性，便于自动化工具提取和处理数据。

硬件规范文档中应包含完整的外设功能描述、寄存器布局和详细描述。功能描述应当清晰地阐述外设的用途、工作模式以及可能的配置选项。寄存器布局则需要详细到每个位的意义、读写权限和可能的值域。此外，文档中还应包含用文字和列表形式描述的操作例程，这些例程可以帮助开发者理解如何通过编程来实现特定的硬件操作。

值得注意的是，虽然图片和图表可以增强人类开发者的理解和记忆，但它们对于自动化工具来说通常是难以解析的。因此，硬件规范文档中不应包含图片，除非这些图片是通过某种图描述语言（如 SVG）创建的，以便供自动化工具解析，或渲染为图形供人类阅读。这样可以确保自动化工具能够准确地从文档中提取所有必要的信息，生成准确无误的驱动代码。

为了充分发挥自动化外设驱动开发工具的潜力，我们呼吁硬件厂商在发布新外设时，能够同时提供符合上述标准的硬件规范文档。提供这样的规范文档将是硬件厂商对开发者社区的一种重要支持。通过这种方式，硬件厂商不仅能够加快产品的市场推广速度，还能塑造活跃开放的企业形象，在开发者、用户和市场中树立良好的声誉。规范文档不仅能够帮助开发者更快地掌握新技术，还能够促进更多的创新和定制解决方案的出现。当开发者能够轻松访问和理解硬件的详细信息时，他们将能够更有效地开发出与硬件紧密集成的软件，从而提升整个系统的性能和可靠性。最终，这种开放和标准化的文档提供方式将有助于建立一个更加健康和协作的嵌入式系统生态系统。硬件厂商、软件开发者和最终用户都将从中受益，共同推动技术的快速发展和创新。
