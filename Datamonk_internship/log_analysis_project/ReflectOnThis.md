# ReflectOnThis.md

A helpful guide to understanding core Linux commands and best practices as a beginner. Includes examples, use cases, and safety tips.

---

## 🆘 Getting Help in Linux

Linux provides **three main ways** to get help about commands. These exist because not all commands are created the same way — some are **built into the shell**, and others are **external programs**.

| Command        | What It Does                           | Best For...                                 |
|----------------|-----------------------------------------|----------------------------------------------|
| `man <command>`| Opens a full manual page                | Learning all options of a command like `ls`  |
| `<command> --help` | Shows a quick summary              | Quickly checking available flags             |
| `help <command>` | Shows help for bash built-in commands | Commands like `cd`, `exit`, `history`        |

### ✅ Examples:

- Full manual:
  ```bash
  man ls
