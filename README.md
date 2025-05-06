# SAM (SSH Alias Manager) v1.0.0

`SAM` (SSH Alias Manager) is a lightweight CLI tool for managing SSH aliases in your `~/.ssh/config` file with ease. It provides interactive prompts, flag-based commands, IP/hostname validation, and SSH key permission checks. Out of the box it includes Zsh tab-completion and a man page.

## Features

* **List aliases** with IP/hostname: `sam -ls`
* **Add alias** (interactive or via flags): `sam --add [-h IP] [-a ALIAS] [-u USER] [-i KEY_PATH]`
* **Edit alias**: `sam --edit -a ALIAS`
* **Remove alias**: `sam --remove -a ALIAS`
* **Validation** for IP format and SSH key existence & permissions
* **Zsh autocompletion** support
* **Man page** (`sam(1)`) for detailed usage reference
* Packaged as a **Debian `.deb`** for easy install (v1.0.0)

## Installation

### From Debian package v1.0.0

Download the `sam_1.0.0_all.deb` from the [Releases](https://github.com/Haitham-7/SAM/releases/tag/v1.0.0) page and install:

```bash
sudo dpkg -i sam_1.0.0_all.deb
sudo apt-get install -f    # if any dependencies are missing
```

### From source

```bash
git clone https://github.com/Haitham-7/SAM.git
cd SAM/sam_1.0.0
sudo dpkg -i sam_1.0.0_all.deb
```

> Or copy the `sam` script to `~/bin` and ensure `~/bin` is in your `$PATH`.

## Usage

Display help:

```bash
sam --help
```

List all aliases with their hostnames/IPs:

```bash
sam -ls
```

Add a new alias (interactive):

```bash
sam --add
```

Add via flags:

```bash
sam --add -h 192.168.1.10 -a web-vm -u ubuntu -i ~/.ssh/id_rsa
```

Edit an existing alias:

```bash
sam --edit -a web-vm
```

Remove an alias:

```bash
sam --remove -a web-vm
```

## Zsh Autocompletion

The `_sam` completion script is installed to `/usr/share/zsh/site-functions/`. To enable it:

```bash
autoload -U compinit && compinit
```

Pressing `TAB` after `sam --` will list available flags.

## Man Page

View the manual with:

```bash
man sam
```

## Contributing

I'll try to maintain it if I can, but don't get your hopes up, your best bit is to fork and edit yourself :) Have Fun! 


## License

This project is licensed under the **GNU General Public License v3.0**. See [LICENSE](LICENSE) for full license text.

You can freely use, modify, and redistribute this software under the terms of the GPLv3, which requires that any derivative works also be open-source under the same license.
