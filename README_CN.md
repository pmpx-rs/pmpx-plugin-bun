# pmpx-plugin-bun

> English see [README.md](README.md)

[pmpx](https://crates.io/crates/pmpx) 的 **bun** 后端。它只做一件事：把 pmpx 的动词映射成 bun
命令 —— 不读文件、不看环境变量、不联网。

```console
$ pmpx install react        # bun 项目里 → bun add react
$ pmpx build --outdir dist  #            → bun run build --outdir dist
$ pmpx exec vite            #            → bunx vite
```

## 映射表

| pmpx | bun |
| ---- | --- |
| `install` | `bun install` |
| `install <pkg>...` | `bun add <pkg>...` |
| `remove <pkg>...` | `bun remove <pkg>...` |
| `run <script> [arg]...` | `bun run <script> [arg]...` |
| `build [arg]...` | `bun run build [arg]...` |
| `test [arg]...` | `bun run test [arg]...` |
| `update` | `bun update` |
| `update <pkg>...` | `bun update <pkg>...` |
| `exec [arg]...` | `bunx [arg]...` |

## `build` 和 `test` 必须走 `run`

这是这个后端唯一需要知道的事，而且它不是风格问题。

**`bun build` 和 `bun test` 是 bun 自己的内置命令**（一个打包器、一个测试运行器），会**遮蔽**
`package.json` 里的同名脚本。省掉 `run` 不是"脚本以略微不同的参数被跑起来"，而是**脚本根本
不会被跑到**。

实测，`package.json` 里的 `build` / `test` 脚本会回显收到的参数：

```console
$ bun build --foo
error: Missing entrypoints. What would you like to bundle?
                              # ↑ bun 的打包器，不是 package.json 的 build 脚本  ✗

$ bun run build --foo
SCRIPT GOT:--foo              # ↑ 脚本跑起来了，参数也拿到了                    ✓

$ bun test --foo
                              # ↑ bun 的测试运行器跑了，脚本根本没被跑到        ✗

$ bun run test --foo
SCRIPT GOT:--foo              # ↑ 脚本跑起来了                                  ✓
```

`build` 那条至少会大声报错。`test` 那条不会：`bun test` 会自顾自地跑完 bun 的测试发现并给出
结果，所以一个 `test` 脚本干别的事的项目会在毫无提示的情况下跑到错的东西。因此这两条一律映射成
`bun run build` 和 `bun run test`。

`install` / `add` / `remove` / `update` / `bunx` 不受影响 —— 它们就是 bun 自己的命令、
做的正是动词的意思，也没有脚本会遮蔽它们。

## 不插 `--`

和那些"不认识的参数就拒绝"的工具不同，bun 会把自己不认识的参数转发给脚本，而且本来就带 `--`
时它会剥掉。用一个回显参数的脚本实测：

```console
$ bun run probe --foo      → GOT:--foo
$ bun run probe -- --foo   → GOT:--foo
```

两种都行，所以这个后端什么都不插：pmpx 透传什么，bun 就看见什么。

## 安装

```console
$ pmpx plugin add bun
```

每次发版还会为常见 target（Linux x64、Windows x64、两种 macOS 架构）上传 prebuilt 产物。
`crate-plugin-kit` 会从同一个 tag 的 release 下载，因此安装通常是一秒而不是一次编译；
没有产物的 target 会回落到从源码编译 —— 只是慢，不是不能用。

## 检测

依据随这个 crate 一起发布的 `pmpx-plugin.toml`：

| 文件 | 权重 | 能证明什么 |
| ---- | ---- | ---------- |
| `bun.lock` | 强（100） | 这个项目确实被 bun 解析过（当前，文本格式） |
| `bun.lockb` | 强（100） | 同上，旧的二进制格式 |
| `package.json` | 弱（10） | 只证明属于这个生态，不证明用了哪个工具 |

两个锁文件名都要列，因为 bun 改过锁文件名：没迁过的项目手里只有 `bun.lockb`，而它和
`bun.lock` 一样是证据。`package.json` 弱，是因为在这个生态里每个 Node 包管理器都会声称它。

## 环境要求

Rust **1.82+**，这是契约 crate 的地板。

## 仓库

<https://github.com/pmpx-rs/pmpx-plugin-bun>

## 许可

MIT —— 见 [LICENSE](LICENSE)。
