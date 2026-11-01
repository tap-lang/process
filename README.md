# process

在 tap 里执行子进程并**拿到输出**。对标 Rust 的 `std::process::Command`。

`std.sys.system()` 只返回退出码，`popen` 只能单向且拿不到分离的 stderr ——
这个包用 `fork + execvp + 两个 pipe` 把 stdout / stderr 分开捕获。

## 用法

```tap
import command;

fn main(): i32 {
    let cmd: Command = command.new("sh");
    cmd = command.arg(cmd, "-c");
    cmd = command.arg(cmd, "echo out; echo err >&2; exit 3");
    cmd = command.env(cmd, "LC_ALL", "C");
    cmd = command.current_dir(cmd, "/tmp");

    let result: Result<Output, String> = command.output(cmd);
    match (result) {
        Result.Ok(value) => {
            print("exit=%d\n", value.status);
            print("out=%s", value.stdout);
            print("err=%s", value.stderr);
        }
        Result.Err(reason) => {
            print("failed: %s\n", reason);
        }
    }
    return 0;
}
```

注意写法是 `command.arg(cmd, "-c")` 而不是 `cmd.arg("-c")`：tap 目前不支持在
表达式上调用方法，只有标识符接收者可以（`arr.push(x)` 行，`obj.field.push(x)` 不行）。

## API

| 函数 | 说明 |
|---|---|
| `new(program)` | 构造。`program` 按 `PATH` 查找（`execvp` 语义） |
| `arg(cmd, value)` | 追加一个参数 |
| `args(cmd, values)` | 追加一组参数（`Vec<String>`） |
| `env(cmd, key, value)` | 设置环境变量（覆盖已有值） |
| `current_dir(cmd, path)` | 子进程的工作目录 |
| `output(cmd)` | 执行并等待，返回 `Result<Output, String>` |
| `status_inherit(cmd)` | 执行并等待，继承标准流以实时显示子进程输出，返回退出码 |
| `status(cmd)` | 捕获并丢弃输出，只返回退出码 |

`Output` 有三个字段：`status: i32`（退出码）、`stdout: String`、`stderr: String`。

`Result.Err` 只覆盖**启动失败**（`pipe()` / `fork()` 失败）。命令不存在不算 Err ——
`execvp` 失败时子进程按约定 `_exit(127)`，你会在 `status` 里看到 127。

## 限制

- **不支持 stdin 重定向**。子进程继承父进程的 stdin。
- **信号**：被信号杀死的进程 `status` 返回 `-1`（Rust 返回 `None`，tap 没有 Option）。
- **仅 POSIX**。`fork` / `execvp` / `pipe` 在 Windows 上不存在。

（大输出死锁的问题已经用 `poll` 解决了——见下面「为什么用 poll」。）

## 实现要点

- **为什么用 `poll` 同时读两路**。顺序读（先读完 stdout 再读 stderr）在子进程往 stderr
  写超过管道缓冲（通常 64 KB）时会死锁：子进程阻塞在 `write`，父进程还在等 stdout 的
  EOF。`poll` 告诉我们哪一路可读，一次循环把可读的都读掉。
  实测 stderr 输出 175 KB 也能正常读完，不死锁。
- **所有 C 字符串在 `fork` 之前构造**。实测在子进程里构造会拿到乱码 ——
  `fork` 之后的内存状态不可靠。所以 `program` / `argv` / `env` 全部在父进程侧一次建好。
- **`argv` 用 `Vec<*u8>` 而不是 `malloc` 出来的 `**u8`**。`env_names[i]` 这种下标在
  tap 里推断出来会少一层指针（和 `Vec<*T>` 那个 bug 同源），`vec.get` 的返回类型才准。
  另外所有 `push` 做完再取 `.data` —— 中途 realloc 会让之前拿到的指针失效。
- **`String` 转 C 字符串是逐字节复制**。`__tap_string_copy_bytes` 这个 Runtime ABI
  已经被 `std.byte_vec` 声明过，顶层函数名全程序唯一，再声明一次会报 duplicate。
  命令名和参数都很短，逐字节的开销可以忽略。
- **`_exit` 声明成返回 `i32`**。tap 没有 `void` 返回类型，而 `_exit` 实际不返回；
  声明成 `i32` 只是让调用点通过类型检查。子进程里必须用 `_exit` 而不是 `return` ——
  `return` 会继续执行下面的父进程代码。

## 运行演示

```sh
tap run src/main.tp
```

输出（六个场景）：捕获 stdout、分离 stdout/stderr、工作目录、环境变量、
命令不存在（exit 127）、只看退出码。
