#  SAM (SSH Alias Manager)

`SAM` is a simple, self-contained **Bash CLI tool** for managing SSH aliases directly from your `~/.ssh/config`.  
It supports **listing**, **adding**, **editing**, **removing**, and **connecting** to aliases, all with smart parsing, validation, and safety features.

---

##  Features

 ### **Smart SSH Config Expansion**  
  Expands `Include` directives automatically into a temporary merged config for accurate parsing.

 ### **Rich, Tabular Listing**  
  Displays alias, host, user, key path, and port in a formatted table with automatic indexing.

 ### **Index-based SSH Access**  
  Connect instantly without typing the alias name.

 ### **Add / Edit / Remove Aliases**  
  - Fully interactive or CLI flag-based control  
  - Automatically backs up your SSH config before every modification  
  - Prevents unsafe edits to included configs

 ### **Additional Features**  
  - Creates timestamped backups (`~/.ssh/config.bak.<timestamp>`)  
  - Validates host/IP format  
  - Checks key existence and permissions (`chmod 600`)  
  - Offers auto-fix for insecure keys
  - Honors `$SAM_CONFIG` for custom config paths  
  - Overrides included file entries safely by appending new Host blocks

---

##  Installation

From source:
```bash
git clone https://github.com/Haitham-7/SAM.git
cd SAM
chmod +x sam
sudo mv sam /usr/local/bin/sam   # or put it anywhere on your $PATH
```

##  Usage

List all aliases with effective details + indices
```bash
sam ls
```
SSH into the 3rd alias from the list
```bash
sam ssh 3
```
Add alias (interactive)
```bash
sam add
```
Add alias via flags
```bash 
sam add -h 192.168.1.10 -u ubuntu -i ~/.ssh/id_rsa -a web-server
```
Edit alias by name (interactive)
```bash
sam edit -a web-server
```
Edit alias by index (interactive)
```bash
sam edit 2
```
Remove alias by name
```bash
sam remove -a web-server
```
Remove alias by index
```bash
sam remove 2
```

---

## Contributing

I'll try to maintain it if I can, but don't get your hopes up. Your best option is to fork and edit yourself. Have Fun!


## License

This project is licensed under the **GNU General Public License v3.0**. See [LICENSE](LICENSE) for full license text.

You can freely use, modify, and redistribute this software under the terms of the GPLv3, which requires that any derivative works also be open-source under the same license.
