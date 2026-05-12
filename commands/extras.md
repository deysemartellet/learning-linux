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

- **Observação sobre flags juntas:** Sempre que houver flags juntas como `-antp` (no `netstat`) ou `-aux` (no `ps`), são apenas várias flags individuais “abraçadas” para o comando ficar curto.

No caso do `lsoft`, o padrão `-i -nP` é o mais usado por analistas de segurança justamente para não ter erro; primeiro diz que quer ver a **Internet (`-i`)** e depois diz que quer os **Números Puros (`-nP`)**.

No caso do `lsoft`, o padrão `-i -nP` é o mais usado por analistas de segurança justamente para não ter erro; primeiro diz que quer ver a **Internet (`-i`)** e depois diz que quer os **Números Puros (`-nP`)**.


### Modify vs. Change (file-analysis)

- Modify (mtime - Modification Time) refere-se ao **conteúdo** do arquivo
  
  - **O que aciona:** Abrir o arquivo (imaginando um arquivo `.txt`).
    
  - **Analogia:** Abrir uma caixa e trocar o que estava dentro dela.
    
  - **Comando de exemplo:** `echo “texto novo” >> arquivo.txt`
 
    
- Change (ctime - Change Time) refere-se aos **metadados** (as propriedades da “caixa”)

  - **O que aciona:** Mudar as permissões (`chmod`), mudar o dono (`chown`), renomear o arquivo ou mover para outro diretório.
    
  - **Detalhe Crucial:** Toda vez que o **Modify** muda, o **Change** também muda automaticamente (pois o tamanho do arquivo, que é um metadado, foi alterado). Mas o contrário não é verdadeiro: podemos mudar o dono do arquivo (Change) sem nunca tocar no conteúdo dele (Modify).
  - **Analogia:** Pintar o lado de fora da caixa ou trocar o cadeado, mas não tocar no que está dentro.
    
  - **Comando de exemplo:** `chmod +x arquivo.txt`


#### Por que o SOC se importa?

O campo Modify pode ser facilmente falsificado (o atacante usa o comando `touch -d` e diz ao sistema: “finja que este arquivo é de 2010”).

No entanto, o campo Change é controlado pelo Kernel do Linux. É muito mais difícil para um invasor comum alterar o Change sem ter privilégios altíssimos ou manipular o relógio do sistema, o que torna o Change a data mais confiável em uma investigação forense para saber quando um arquivo realmente “sofreu uma intervenção”.


Ou seja:
  - **Modify** - mudou o que está dentro (falsificável);
  - **Change** - mudou a estrutura/permissão (difícil de falsificar).
