# `tee`



Tee 命令是 Linux/Unix 系统中的一个常用命令，它用于读取标准输入的数据，并将其内容同时输出到标准输出（通常是终端）和一个或多个文件中。

### 主要功能

1. **将输入内容输出到标准输出和文件中：**
   `tee` 命令的最基本用法是将输入的内容同时输出到屏幕和文件中。例如：

   ```shell
   echo "Hello, World!" | tee output.txt
   ```

   这会将 "Hello, World!" 输出到屏幕并写入 `output.txt` 文件。

2. **追加内容到文件：**
   使用 `-a` 选项，可以将内容追加到文件中，而不是覆盖文件的内容。例如：

   ```shell
   echo "Append this line." | tee -a output.txt
   ```

   这会将 "Append this line." 追加到 `output.txt` 文件的末尾。

3. **同时写入多个文件：**
   `tee` 也可以将输出写入多个文件。例如：

   ```shell
   echo "This will be written to two files." | tee file1.txt file2.txt
   ```

4. **与其他命令结合使用：**
   `tee` 常常与其他命令结合使用，通过管道将命令的输出写入文件。例如，将 `ls` 命令的输出同时显示在屏幕上并保存到文件：

   ```shell
   ls -l | tee output.txt
   ```

### 常用选项

- `-a`：将输出追加到文件，而不是覆盖文件。
- `-i`：忽略中断信号。

### 示例

- 将 `ps` 命令的输出写入日志文件并同时在屏幕上显示：

  ```shell
  ps aux | tee process.log
  ```

`tee` 命令在调试、日志记录等场景中非常有用，尤其是在需要同时查看和保存命令输出时。