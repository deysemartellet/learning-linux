## 🎯 Objetivo da análise

Nessa etapa, aprendo que a análise de arquivos, que é o processo de investigar a **integridade**, a **origem** e o **comportamento** de um arquivo para determinar se ele é legítimo ou parte de um incidente.

Em uma investigação, busca-se responder:

- **O que mudou?** (Integridade)

- **Quando mudou?** (Timeline)

- **Quem mudou?** (Propriedade e Permissões)

Para análise de arquivos para SOC, vou aprofundar nos comandos `find`, `ls -lat` e `stat`.

**Por que isso é importante para o SOC?**

Atacantes tentam se esconder (técnica de **Defense Evasion**). Eles podem dar nomes legítimos aos arquivos, mas é difícil alterar todos os metadados sem deixar inconsistências.

A análise desses metadados é, muitas vezes, a única prova que temos de que algo foi violado.

# 🌐 Análise de Arquivos

## 📌 Comando

### find

### O que ele faz
O `find` busca arquivos e diretórios com base em filtros específicos: nome, tamanho, tipo, permissões, dono e, o mais importante para segurança, **data de modificação ou acesso.**

#### Uso em SOC

- **Detecção de Artefatos:** Encontrar scripts novos criados em pastas temporárias (`/tmp`).

- **Integridade:** Localizar arquivos de configuração que foram alterados nos últimos minutos após um alerta.

- **Web Shells:** Buscar por arquivos `.php` ou `.jsp` criados recentemente em servidores web.

- **Binários com SUID:** (Set User ID) Achar arquivos que permitem que um usuário comum rode comandos como `root`.

#### Exemplo
```bash
find /etc
```

### O que cada flag faz

- ##### `find / -name “.php”:` Busca todos os arquivos PHP no sistema.
- ##### `find /tmp -type f -mmin -10`:** Busca arquivos (`-type f`) em `/tmp` modificados nos últimos 10 minutos.
- ##### `find /var/www -user [REDACTED]`: Busca arquivos pertencentes a um usuário específico em uma pasta.
### Flags essenciais para Segurança
- #### **`-mtime -1`:** Arquivos modificados nas últimas **24 horas**.
- #### **`-mmin -60`:** Arquivos modificados nos últimos **60 minutos**.
- #### **`-perm /4000`:** Busca arquivos com o bit SUID (muito perigoso se for um arquivo desconhecido.

## 🧪 Mini cenário: O Script de Persistência

O alerta do SOC indicou uma alteração inesperada no comportamento do servidor às 14:00. Agora são 14:30. Suspeita-se que o atacante deixou um arquivo desconhecido para voltar depois.

### Passos:
1. Executar `sudo find /etc /tmp /var/www -type f -mmin -40 -ls` para ver tudo o que foi alterado nos últimos 40 minutos

#### Output simulado de processo suspeito
```bash
[REDACTED] 4096 Oct 25 14:12 /etc/cron.d/system_update
[REDACTED] 12288 Oct 25 14:15 /var/www/html/assets/style.css.php         

```

#### Investigação dos arquivos
1. `/etc/cron.d/system_update:`: O atacante criou uma tarefa agendada (cron job) para rodar um comando malicioso periodicamente. O nome parece legítimo, mas a **data de modificação** recente entrega a manobra.

2. `/var/www/html/assets/style.css.php`:: Um arquivo `.php` escondido dentro de uma pasta de estilos (`assets`). Isso é um **Web Shell** clássico: o invasor agora pode rodar comandos no servidor via navegador acessando esse arquivo oculto.

3. **Flags usadas:** O `-ls` no final do comando `find` é ótimo porque já mostra detalhes como permissões e data, similar ao `ls -l`.


##### Conclusão e Próximos Passos
No SOC, usamos muito o termo **IoC (Indicator of Compromise)**. Esses arquivos que encontrei com `find` são meus IoCs.

Após confirmar que o arquivo `/var/www/html/assets/style.css.php` é um **Web Shell** e que há uma tarefa agendada suspeita em `/etc/cron.d/system_update`, as ações imediatas seriam:

1. **Quarentena e Preservação:** Antes de deletar, vamos mover os arquivos para um diretório isolado ou mudar as permissões para que não sejam executáveis (`chmod 000`), permitindo uma análise forense posterior se necessário.

2. **Interrupção do Processo:** Se o `find` mostrou que esses arquivos foram modificados por um processo ativo (que identificamos com `lsof` ou `fuser`), esse processo deve ser encerrado imediatamente.

3. **Remoção da Persistência:** Deletar a tarefa no `cron` para evitar que o malware seja baixado ou executado novamente de forma automática.

4. **Auditoria de Logs:** Investigar os logs do servidor web (Apache/Nginx) para descobrir **como** o atacante conseguiu fazer o upload desse arquivo (ex: uma vulnerabilidade em um plugin desatualizado).

5. **Análise de Impacto:** Verificar se o bit **SUID** foi aplicado a outros binários do sistema para garantir que o atacante não deixou outras “portas abertas” com privilégios de root.


## 📌 Comando

### stat

#### O que faz
O `stat` (Status) é um comando que exibe informações detalhadas sobre o estado de um arquivo ou sistema de arquivos, focando especialmente nos seus **metadados** e selos temporais (timestamps).

Ele extrai do sistema de arquivos informações que não são visíveis no conteúdo do arquivo, como o número **Inode** (identificador único do arquivo no sistema de arquivos (como um “RG” do arquivo dentro do disco)), o tamanho exato em blocos, o dono (UID), o grupo (GID) e os três selos temporais principais.

#### Uso em SOC

- **Detecção de Timestomping:**. Atacantes tentam alterar as datas de um arquivo para que ele pareça antigo e legítimo. O `stat` mostra nanosegundos e o selo **Change** é gerenciado pelo kernel e tende a revelar alterações recentes mesmo quando **Access** e **Modify** foram manipulados.

- **Rastreamento de Inode:** Se um arquivo foi deletado e um novo foi criado com o mesmo nome, o número do Inode no `stat` mudará, revelando a substituição.

- **Análise de MAC:** Ver exatamente quando um malware foi executado pela última vez (Access).

#### Exemplo
```bash
stat [REDACTED].php
stat -f /:
```

- `stat [REDACTED].php`: Mostra todos os detalhes do arquivo.
  
- `stat -f /`: Mostra informações sobre o sistema de arquivos (espaço, limites de nomes de arquivos).

#### Os Três Selos Temporais (Essencial para o SOC)

No Linux, o `stat` foca no acrônimo **MAC**:

- **Access (A):** A última vez que o arquivo foi lido (ex: alguém deu um `cat` ou executou o script).

- **Modify (M):** A última vez que o **conteúdo** do arquivo foi alterado.

- **Change (C):** A última vez que os **metadados** (permissões, dono, nome) foram alterados.

## 🧪 Mini cenário: O Arquivo “Viajante do Tempo”


Encontrei um executável suspeito chamado `update_bin` em `/usr/bin`. O atacante é esperto e usou um comando para fazer o arquivo parecer ter sido criado há 2 anos.


### Passos:


1. Executar `stat /usr/bin/update_bin`
2. Observo a seguinte saída:
```
  File: /usr/bin/update_bin
  Size: 45600      Blocks: 96         IO Block: 4096   regular file
Device: 801h/2049d Inode: 1572865     Links: 1
Access: (0755/-rwxr-xr-x)  Uid: ( 0/ root)   Gid: ( 0/ root)
Access: 2022-05-10 10:00:00.000000000 -0300
Modify: 2022-05-10 10:00:00.000000000 -0300
Change: 2024-10-25 15:30:22.456789123 -0300
 Birth: -
```

#### Investigação dos arquivos


1. **A Incoerência:** O **Access** e o **Modify** dizem “2022”. Parecem antigos e seguros, mas não são.

2. **Change:** O selo **Change** mostra “2024-10-25”, porque enquanto um atacante pode facilmente alterar as datas de acesso e modificação com o comando `touch`, o selo **Change** é atualizado pelo kernel sempre que as permissões ou metadados mudam.

3. O que podemos concluir é que alguém “trouxe” esse arquivo de fora ou alterou suas permissões hoje às 15:30. O arquivo é um intruso recente tentando se passar por antigo.


#### Conclusão


Ao encontrar essa discrepância no `stat`, o analista confirma que houve manipulação de linha do tempo (**Timestomping**).


1. **Bloqueio:** O arquivo deve ser movido para análise.
   
2. **Timeline:** Agora sei o horário exato (`15:30:22`) para procurar nos logs o que mais aconteceu naquele segundo.

##

- `ls -lat` = Ordem cronológica (Novos em cima) + Arquivos Ocultos + Detalhes técnicos.


**Obs.:** Sempre olhar quem é o **Dono (Owner)**. Se um arquivo em uma pasta do `root` pertence ao usuário do servidor web (`www-data`), há algo errado.
