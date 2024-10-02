1. 刻盘 做启动盘 ventoy
2. 有的电脑有 bitlocker，解锁磁盘
3. 进 bios
    1. 关闭安全启动 secure boot
    2. 设置启动选项，从 u 盘启动
    3. try or intall ubuntu 黑屏：在 quiet splash --- 替换 --- 为 nomodeset（禁用显卡的启动模式） 
    4. 设置分区方案
4. 如果4090启动卡住，禁用 ubuntu 默认驱动
5. 启动 ubuntu 系统
6. 检查 驱动 禁用 ubuntu 自带驱动
7. apt 换源
8. 下载 cuda 对应版本 并安装

