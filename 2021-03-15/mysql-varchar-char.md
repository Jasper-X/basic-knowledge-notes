# MySQL 中 varchar 和 char 的区别是什么？

- varchar是可变长度的类型，char是不可变长度的类型
- varchar最大长度是65535字节，char的最大长度是255字节
- varchar有1-2字节长度的前缀用于标记列长度，char无前缀
- varchar的存储开销是数据的字节数和前缀字节数，char是数据的字节数和补位的空格数
- varchar查找效率不如char