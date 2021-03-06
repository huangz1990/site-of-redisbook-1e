重点回顾
----------

- Redis 2.8 以前的复制功能不能高效地处理断线后重复制情况，
  但 Redis 2.8 新添加的部分重同步功能可以解决这个问题。

- 部分重同步通过复制偏移量、复制积压缓冲区、服务器运行 ID 三个部分来实现。

- 在复制操作刚开始的时候，
  从服务器会成为主服务器的客户端，
  并通过向主服务器发送命令请求来执行复制步骤，
  而在复制操作的后期，
  主从服务器会互相成为对方的客户端。

- 主服务器通过向从服务器传播命令来更新从服务器的状态，
  保持主从服务器一致，
  而从服务器则通过向主服务器发送命令来进行心跳检测，
  以及命令丢失检测。
