# ch-12

# 第12章：WINDOWS系统交互与分析

开发 Microsoft Windows 攻击的方法数不胜数，本章无法涵盖太多。无论在最初还是后期开发中，我们只介绍并研究几个对攻击Windows有用的技术，并非研究所有的。

我们将从三个主题来讨论后面的Microsoft API 文档和安全问题。第一，使用Go的 `syscall` 包，通过执行进程注入和各系统级的别的Window API交互。第二，探索Go的Windows可执行文件(PE)格式核心包，并编写一个PE文件格式解析器。