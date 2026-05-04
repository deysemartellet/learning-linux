## 🔍 Flow da Triagem
#### Quem está no sistema?
``` bash
who, w, last
```
#### O que está usando?
```bash
ps, top
```
#### O que está conectado?
``` bash
netstat, ss, lsof
```
#### O que mudou?
``` bash
find, stat, ls -lat
```
#### O que os logs dizem?
```bash
grep, /var/log/*
```
#### Há persistência?
``` bash
crontab, systemctl
```
#### Há algo escondido?
``` bash
strings, weird files
```

- Se o `ss` ou o `netstat` mostram **o que** está acontecendo na rede, o `ps aux` diz **quem** é o responsável dentro do sistema. 
Esses dois comandos são como as duas lentes de um binóculo para um analista SOC.

- **Observação sobre flags juntas:** Sempre que houver flags juntas como `-antp (no `netstat`) ou `-aux` (no `ps`), são apenas várias flags individuais “abraçadas” para o comando ficar curto.
No caso do `lsoft`, o padrão `-i -nP` é o mais usado por analistas de segurança justamente para não ter erro; primeiro diz que quer ver a **Internet (`-i`)** e depois diz que quer os **Números Puros (`-nP`)**.
