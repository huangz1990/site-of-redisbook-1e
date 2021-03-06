旧版复制功能的实现
---------------------------

Redis 的复制功能分为同步（sync）和命令传播（command propagate）两个操作：

- 其中，
  同步操作用于将从服务器的数据库状态更新至主服务器当前所处的数据库状态。

- 而命令传播操作则用于在主服务器的数据库状态被修改，
  导致主从服务器的数据库状态出现不一致时，
  让主从服务器的数据库重新回到一致状态。

本节接下来将对同步和命令传播两个操作进行详细的介绍。


同步
^^^^^^^^^^^

当客户端向从服务器发送 :ref:`SLAVEOF` 命令，
要求从服务器复制主服务器时，
从服务器首先需要执行同步操作，
也即是，
将从服务器的数据库状态更新至主服务器当前所处的数据库状态。

从服务器对主服务器的同步操作需要通过向主服务器发送 :ref:`SYNC` 命令来完成，
以下是 :ref:`SYNC` 命令的执行步骤：

1. 从服务器向主服务器发送 :ref:`SYNC` 命令。

2. 收到 :ref:`SYNC` 命令的主服务器执行 :ref:`BGSAVE` 命令，
   在后台生成一个 RDB 文件，
   并使用一个缓冲区记录从现在开始执行的所有写命令。

3. 当主服务器的 :ref:`BGSAVE` 命令执行完毕时，
   主服务器会将 :ref:`BGSAVE` 命令生成的 RDB 文件发送给从服务器，
   从服务器接收并载入这个 RDB 文件，
   将自己的数据库状态更新至主服务器执行 :ref:`BGSAVE` 命令时的数据库状态。

4. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器，
   从服务器执行这些写命令，
   将自己的数据库状态更新至主服务器数据库当前所处的状态。

图 IMAGE_SYNC 展示了 :ref:`SYNC` 命令执行期间，
主从服务器的通信过程。

.. graphviz::

    digraph {

        label = "\n 图 IMAGE_SYNC    主从服务器在执行 SYNC 命令期间的通信过程"

        rankdir = LR

        splines = ortho

        node [shape = box, height = 2]

        master [label = "主\n服\n务\n器"]

        slave [label = "从\n服\n务\n器"]

        master -> slave [dir = back,  label = "发送 SYNC 命令"]

        master -> slave [label = "\n 发送 RDB 文件"]

        master -> slave [label = "\n 发送缓冲区保存的所有写命令"]

    }

表 TABLE_SYNC_EXAMPLE 展示了一个主从服务器进行同步的例子。

+-------+-------------------------------------------+-----------------------------------------------+
| 时间  | 主服务器                                  | 从服务器                                      |
+=======+===========================================+===============================================+
| T0    | 服务器启动。                              | 服务器启动。                                  |
+-------+-------------------------------------------+-----------------------------------------------+
| T1    | 执行 ``SET k1 v1`` 。                     |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T2    | 执行 ``SET k2 v2`` 。                     |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T3    | 执行 ``SET k3 v3`` 。                     |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T4    |                                           | 向主服务器发送 :ref:`SYNC` 命令。             |
+-------+-------------------------------------------+-----------------------------------------------+
| T5    | 接收到从服务器发来的 :ref:`SYNC` 命令，   |                                               |
|       | 执行 :ref:`BGSAVE` 命令，                 |                                               |
|       | 创建包含键 ``k1`` 、 ``k2`` 、 ``k3``     |                                               |
|       | 的 RDB 文件，                             |                                               |
|       | 并使用缓冲区记录接下来执行的所有写命令。  |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T6    | 执行 ``SET k4 v4`` ，                     |                                               |
|       | 并将这个命令记录到缓冲区里面。            |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T7    | 执行 ``SET k5 v5`` ，                     |                                               |
|       | 并将这个命令记录到缓冲区里面。            |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T8    | :ref:`BGSAVE` 命令执行完毕，              |                                               |
|       | 向从服务器发送 RDB 文件。                 |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T9    |                                           | 接收并载入主服务器发来的 RDB 文件 ，          |
|       |                                           | 获得 ``k1`` 、 ``k2`` 、 ``k3`` 三个键。      |
+-------+-------------------------------------------+-----------------------------------------------+
| T10   | 向从服务器发送缓冲区中保存的写命令        |                                               |
|       | ``SET k4 v4`` 和 ``SET k5 v5`` 。         |                                               |
+-------+-------------------------------------------+-----------------------------------------------+
| T11   |                                           | 接收并执行主服务器发来的两个 :ref:`SET` 命令，|
|       |                                           | 得到 ``k4`` 和 ``k5`` 两个键。                |
+-------+-------------------------------------------+-----------------------------------------------+
| T12   | 同步完成，                                | 同步完成，                                    |
|       | 现在主从服务器两者的数据库都包含了键      | 现在主从服务器两者的数据库都包含了键          |
|       | ``k1`` 、 ``k2`` 、 ``k3`` 、 ``k4`` 和   | ``k1`` 、 ``k2`` 、 ``k3`` 、 ``k4`` 和       |
|       | ``k5`` 。                                 | ``k5`` 。                                     |
+-------+-------------------------------------------+-----------------------------------------------+


命令传播
^^^^^^^^^^^

在同步操作执行完毕之后，
主从服务器两者的数据库将达到一致状态，
但这种一致并不是一成不变的 —— 
每当主服务器执行客户端发送的写命令时，
主服务器的数据库就有可能会被修改，
并导致主从服务器状态不再一致。

举个例子，
假设一个主服务器和一个从服务器刚刚完成同步操作，
它们的数据库都保存了相同的五个键 ``k1`` 至 ``k5`` ，
如图 IMAGE_CONSISTENT 所示。

.. graphviz::

    digraph {

        label = "\n 图 IMAGE_CONSISTENT    处于一致状态的主从服务器"

        rankdir = LR

        node [shape = record, width = 2]

        subgraph cluster_master {

            label = "主服务器"

            master_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k3> k3 | <k4> k4 | <k5> k5 "];

        }

        
        subgraph cluster_slave {

            label = "从服务器"

            slave_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k3> k3 | <k4> k4 | <k5> k5 "];

        }

        master_db -> slave_db [style = invis]
    }

如果这时，
客户端向主服务器发送命令 ``DEL k3`` ，
那么主服务器在执行完这个 :ref:`DEL` 命令之后，
主从服务器的数据库将出现不一致：
主服务器的数据库已经不再包含键 ``k3`` ，
但这个键却仍然包含在从服务器的数据库里面，
如图 IMAGE_INCONSISTENT 所示。

.. graphviz::

    digraph {

        label = "\n 图 IMAGE_INCONSISTENT    处于不一致状态的主从服务器"

        rankdir = LR

        node [shape = circle]

        client [label = "客户端"]

        node [shape = record, width = 2]

        subgraph cluster_master {

            label = "主服务器"

            master_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k4> k4 | <k5> k5 "];

        }

        
        subgraph cluster_slave {

            label = "从服务器"

            slave_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k3> k3 | <k4> k4 | <k5> k5 "];

        }

        master_db -> slave_db [style = invis]

        client -> master_db [label = "发送命令 \n DEL k3"]
    }

为了让主从服务器再次回到一致状态，
主服务器需要对从服务器执行命令传播操作：
主服务器会将自己执行的写命令 —— 也即是造成主从服务器不一致的那条写命令 —— 发送给从服务器执行，
当从服务器执行了相同的写命令之后，
主从服务器将再次回到一致状态。

在上面的例子中，
主服务器因为执行了命令 ``DEL k3`` 而导致主从服务器不一致，
所以主服务器将向从服务器发送相同的命令 ``DEL k3`` ：
当从服务器执行完这个命令之后，
主从服务器将再次回到一致状态 —— 
现在主从服务器两者的数据库都不再包含键 ``k3`` 了，
如图 IMAGE_PROPAGATE_DEL_k3 所示。

.. graphviz::

    digraph {

        label = "\n 图 IMAGE_PROPAGATE_DEL_k3    主服务器向从服务器发送命令"

        rankdir = LR

        node [shape = record, width = 2]

        subgraph cluster_master {

            label = "主服务器"

            master_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k4> k4 | <k5> k5 "];

        }
        
        subgraph cluster_slave {

            label = "从服务器"

            slave_db [label = " <head> 数据库 | <k1> k1 | <k2> k2 | <k4> k4 | <k5> k5 "];

        }

        master_db -> slave_db [label = "发送命令 \n DEL k3"]
    }

..
    命令传播的注意事项
    ^^^^^^^^^^^^^^^^^^^^^^^

    关于命令传播有三个需要注意的地方。

    首先，
    命令传播是一个持续的过程，
    而不是一个单一的操作：
    只要主从服务器处于正常的连接状态，
    命令传播就会一直进行下去，
    使得主从服务器一直保持一致状态。

    其次，
    命令传播针对的是所有从服务器，
    而不是某个单一的从服务器：
    如果有多个从服务器在复制同一个主服务器，
    那么主服务器在执行写命令之后，
    所有从服务器都将收到主服务器发来的写命令。

    举个例子，
    图 IMAGE_PROPAGATE_COMMAND 展示了一个主服务器在接收到写命令 ``SET key value`` 之后，
    向它的三个从服务器发送相同写命令的过程。

    .. graphviz::

        digraph {

            rankdir = LR;

            node [shape = circle];

            client [label = "客户端", width = 1.47];

            master [label = "主服务器"];

            slave1 [label = "从服务器\nA"];

            slave2 [label = "从服务器\nB"];

            slave3 [label = "从服务器\nC"];

            //

            edge [label = "发送命令 \n SET key value"]

            client -> master

            master -> slave1
            master -> slave2
            master -> slave3

            label = "\n图 IMAGE_PROPAGATE_COMMAND    主服务器向所有从服务器传播命令";
        }

    最后，
    虽然本节一直使用 :ref:`SET` 命令来作为命令传播的示例，
    但除了 :ref:`SET` 命令之外，
    其他写命令 —— 比如 :ref:`RPUSH` 、 :ref:`SADD` 、 :ref:`DEL` 、 :ref:`PUBLISH` 、 :ref:`EXPIRE` 、 :ref:`EVAL` 、 :ref:`EVALSHA` ，
    等等，
    都会被主服务器传播至从服务器。

    .. topic::  注意

        :ref:`PUBLISH` 这类命令虽然并不写入数据库，
        但是却会对客户端产生副作用，
        所以这些命令也会被视为是写命令，
        并被主服务器传播至从服务器。
