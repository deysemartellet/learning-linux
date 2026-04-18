## 🎯 Objetivo da análise

Nesta etapa, buscamos identificar comportamentos anormais relacionados a processos em execução no sistema.

# 🧠 Análise de Processos

## 📌 Comandos

### ps aux

#### O que faz
Lista todos os processos ativos no sistema.
Simplificando: é um “snapshot” (uma foto instantânea) de todos os processos rodando no sistema.

#### Uso em SOC
Identificar processos suspeitos, malware ou execução não autorizada.

#### Exemplo
```bash
ps aux
```

### O que cada flag faz

##### `a` (all): Mostra processos de todos os usuários, não apenas os seus.
##### `u` (user): Exibe o nome do usuário que iniciou o processo e detalhes de consumo (CPU/Memória).
##### `x`: Mostra processos que não estão ligados a um terminal (muito importante, pois a maioria dos malwares e serviços de sistema rodam sem terminal).

### 🔗 Conexão com outros comandos

O PID identificado aqui pode ser investigado com:
- netstat / ss (para ver conexões)


O que observar
Processos desconhecidos
Execução como root
Alto uso de CPU/memória

## 🧪 Mini cenário: O Processo Camaleão

Em um ambiente controlado de simulação de um SIEM, foi recebido um alerta de **alto uso de CPU**. 

### Passos:
1. Executar `ps aux`
2. Identificar processo suspeito
3. Investigar

Executando o comando `ps aux`, vê-se estas colunas principais:
```bash
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  168k  245k ?        Ss   Oct10   0:02 /sbin/init
root      1234 98.2  0.2  4500  1200 ?        S    10:15  45:00 [kworker/1:2]
www-data  4321  0.1  0.5 120k   500k ?        S    11:00   0:00 /usr/sbin/apache2
```
#### Investigação de PID
**A Coluna Command (O disfarce)**: 
O processo `PID 1234` chama-se `[kworker/1:2]`. Processos entre colchetes `[ ]` geralmente são threads do kernel.

**O Alerta**: Threads reais do kernel quase nunca consomem 98% de CPU por tanto tempo. Além disso, malwares costumam usar nomes entre colchetes para que o analista pense que é apenas o sistema operacional trabalhando.

**A Coluna STAT (O estado)**:
`S` significa *Sleep* (dormindo), `R` é *Running* (rodando). Se um processo está sempre em `R` com CPU alta, ele está executando cálculos intensos (como mineração de cripto ou criptografia de arquivos/Randomware).

- **Obs.:** `z` (Zombie): Se houver um processo com `STAT Z`, ele é um processo “Zumbi”. Ele já morreu, mas o processo pai não o removeu da tabela. Em um incidente, muitos processos zumbis podem indicar que um malware tentou rodar vários comandos que falharam ou foram bloqueados por uma ferramenta de segurança (alerta crítico para um analista!)

**A Coluna USER (Privilégios)**:
Se um processo suspeito está rodando como `root`, o perigo é máximo. Se for `www-data` (usuário do servidor web), pode ser um site vulnerável que foi invadido.


##### Dicas observadas durante o cenário:
**Ordernar por CPU**: Para ver quem está “derretendo” o processador agora:
```bash
ps aux –sort=-%cpu | head
```
(O sinal de menos antes do `%cpu` coloca em ordem **decrescente**).

**Ver a “árvore” de processos**: Às vezes o processo pai é o culpado. Use `ps faux` para ver quem chamou quem (a hierarquia).

**Ordenar por RAM**: Enquanto ataques de **mineração de cripto** ou **ataques de força bruta** derretem o **%CPU**, outros tipos de ameaças preferem a **memória (%MEM)**:
```bash
ps aux –sort=-%mem | head
```
Malware sem arquivo: alguns malwares rodam exclusivamente na memória RAM para não deixar rastros no disco rígido. Eles podem fazer o uso de memória subir de forma constante.
Vazamento de Dados (Buffer Overflow): falhas em softwares legítimos que um atacante tenta explorar podem causar picos anormais de uso de memória antes do programa “crashar”.
In-Memory Databases: às vezes, um banco de dados legítimo (como Redis) pode estar sendo abusado para armazenar dados roubados temporariamente antes da exfiltração.

When you do ps aux, pay attention to:

USER ✔️ 

PID ✔️ 

%CPU ✔️ 

%MEM ✔️ 

COMMAND ✔️ 


## 📌 Comandos

### top (Table of Processes)

#### O que faz
Exibe uma lista dinâmica dos processos em execução e atualiza essas informações a cada poucos segundos. Também mostra o estado geral da saúde do sistema (Uptime, Load Average, Memória Livre).

#### Uso em SOC
Identificar picos de processos que aparecem e desaparecem rapidamente, verificar se o sistema está sob um ataque DoS local ou exaustão de recursos, monitorar mudança de PID (sinal de um script que se auto-reinicia).

#### Exemplo
```bash
top
```
### Teclas de Atalho

##### `M`: Ordena por uso de Memória.

##### `P`: Ordena por uso de CPU.

##### `k`: Permite “matar” (kill) um processo fornecendo o PID.

##### `q`: Sai do comando.


### O que cada flag faz

##### `top -u [USER]`: Mostra apenas processos de um usuário específico (ex: `top -u www-data`).
##### `top -p [PID]`: Monitora apenas um processo específico em tempo real.
## 🧪 Mini cenário: O Minerador Silencioso

Em um ambiente controlado de simulação de um SIEM, foi recebido um alerta de que o servidor de banco de dados está extremamente lento.

### Passos:
1. Executar `top`
2. Observar o **Load Average** no topo: Nesse caso, foi visualizado `12.5, 10.1, 9.5`. (Para um servidor de 4 núcleos, isso é altíssimo).
3. Investigar

```bash
PID  USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
5567 [REDACTED] 20   0  154020  12540   2140 R  99.8   0.2  45:12.10 systemd-worker
```

#### Investigação
**Nome do Comando**: `systemd-worker`. Parece um nome legítimo do sistema, mas o `systemd` real raramente usa esse nome exato para tarefas intensas de CPU.

**%CPU em 99.8**: O processo está usando um núcleo inteiro sozinho de forma constante. Isso é comportamento clássico de um **Cryptojacker** (minerador de criptomoedas) escondido sob um nome falso.

**TIME+**: O processo está rodando há mais de 45 minutos sem parar.

#### Conclusão do Cenário
Foi identificado o PID `5567` como a causa da lentidão. O próximo passo lógico seria investigar de onde esse arquivo veio.

## 📌 Comandos

### htop

#### O que faz
É um visualizador de processos interativo e em modo de texto para sistemas **Unix**. Oferece interface colorida, amigável e com suporte a mouse (se o terminal permitir).

Monitora recursos do sistema em tempo real, mostrando o uso de cada **núcleo da CPU** individualmente, possui barras gráficas para memória e swap, e permite rolar a lista de processos horizontal e verticalmente.

#### Uso em SOC
Visualização de “threads” (sub-processos) (malwares modernos costumam se dividir em várias threads para distribuir o peso do processamento), gestão de processos (poder selecionar um processo e enviar um sinal com apenas uma tecla, sem precisar digitar o PID manualmente), além de exibir o caminho completo do executável, facilitando a identificação imediata de arquivos em locais suspeitos.

#### Exemplo
```bash
htop
```
### Teclas de Atalho
No `htop`, as teclas de função (F1-F10) são as principais:

##### `F3` (Search): Busca um processo pelo nome.

##### `F4` (Filter): Filtra a lista (ex: digitar “python” para ver apenas processos Python).

##### `F5` (Tree): Mostra a hierarquia (quem é o processo pai).

##### `F9` (Kill): Abre o menu de sinais para encerrar o processo selecionado.


### Flags úteis

##### `htop -u [USER]`: Monitora apenas processos de um usuário específico (ex: `htop -u [REDACTED]`).
##### `htop -d 10`: Altera o tempo de atualização (delay) para 1 segundo (o padrão é maior), útil para flagrar processos que “piscam” na tela.
## 🧪 Mini cenário: A Árvore de Processos Suspeita

Monitorando um servidor web, percebe-se estar sofrendo uma lentidão intermitente.

### Passos:
1. Abrir o `htop`
2. Aperta `F5` para ver a visualização em árvore.
3. Observa o seguinte cenário:

```bash
  CPU[|||||||||||||||||||||||||98.0%]   Tasks: 45, 120 thr; 1 running
  Mem[|||||||||||            1.2G/8G]   Load average: 1.50 1.20 0.80

  PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
 1043 www-data   20   0  250M  50M    10M S  0.5  0.6  0:05.12 /usr/sbin/apache2
 5678  \_ www-data 20   0  120M  10M  2048 S  0.0  0.1  0:00.01  \_ /bin/sh -c /tmp/data_sync
 5679      \_ www-data 20  0 800M 200M 1024 R 97.5  2.5  10:12.45      \_ /tmp/data_sync --silent
```

#### Investigação

[em progresso]
