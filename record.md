# 2024-11-27

1. 将numberOfThreads=4修改为numberOfThreads=8后，仿真执行时间不降反增，从10m33.316s增加至12m40.713s。

# 2024-11-28

1. 63个逻辑进程（LP）共享4个物理进程。
2. MultithreadedSimulatorImpl接管了NS3默认的模拟器实现SimulatorImpl。
3. 63个逻辑进程加上一个公共逻辑进程，每个逻辑进程都有自己的调度器scheduler



m_mailbox是干什么用的？