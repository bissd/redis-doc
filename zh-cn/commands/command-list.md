返回服务器命令名称的数组。

您可以使用可选的_FILTERBY_修饰符来应用以下过滤器之一：

- **MODULE 模块名称**: 获取属于指定模块的命令。
- **ACLCAT 类别**: 获取 [ACL 类别](/docs/management/security/acl/#command-categories) 中指定类别的命令。
- **PATTERN 模式**: 获取与给定类似 glob 的模式匹配的命令。
