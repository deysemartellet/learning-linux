## 🎯 Objetivo da análise

Nesta etapa, buscamos identificar conexões suspeitas saindo ou entrando da máquina.
**OBS.:** Em sistemas Linux modernos, o comando `netstat` está sendo substituído pelo `ss`. O equivalente moderno seria o comando `ss -antp`.

# 🌐 Análise de Conexões de Rede

## 📌 Comando

### netstat -antp

#### Uso em SOC
Identificar conexões suspeitas, comunicação com IPs externos e possíveis backdoors.

#### Exemplo
```bash
netstat -antp
```

### O que cada flag faz

##### `-a` **(all)**: Exibe todas as conexões ativas e portas que o computador está "ouvindo" (listening).
##### `-n` **(numeric)**: Mostra endereços IP e números de portas em formato numérico, sem tentar resolver nomes de host (como "google.com"), o que torna o comando muito mais rápido.
##### `-t` **(tcp)**: Filtra os resultados para exibir apenas conexões que utilizam o protocolo TCP.
##### `-p` **(programs)**: Mostra o **PID** (ID do processo) e o nome do programa/serviço responsável pela conexão. 
#### **Obs:** *Geralmente requer **sudo** para ver todos os nomes de programas.*
### Uso em SOC
Identificar conexões suspeitas saindo ou entrando na máquina.
## 🧪 Mini cenário: Servidor Invadido

Imaginando um cenário de **servidor invadido** onde um atacante subiu um “backdoor” (porta dos fundos) para manter acesso permanente ao sistema.

### Passos:
1. Executar `sudo netstat -antp`
2. Identificar processo suspeito
3. Investigar PID

#### Output simulado de processo suspeito
```bash
# sudo netstat -antp
Conexões Internet Ativas (servidores e estabelecidas)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      912/sshd            
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1043/apache2        
tcp        0      0 0.0.0.0:6666            0.0.0.0:*               LISTEN      3122/nc             
tcp        0      0 192.168.1.10:80         200.150.10.5:54212      ESTABLISHED 1043/apache2        
tcp        0    256 192.168.1.10:44321      45.79.12.11:80          ESTABLISHED 3125/sh             

```

#### Investigação de PID
**Escuta (LISTEN)**: Na linha do **PID 3122**, o comando `nc` (netcat) está abrindo a porta **6666**. Isso geralmente indica que alguém abriu uma brecha para entrar no sistema a qualquer momento.
**Execução como Root**: Ao rodar com `sudo`, percebe-se que o processo 3125 é apenas um `sh` (shell). Um shell (`sh` ou `bash`) estabelecendo uma conexão de rede (`ESTABLISHED`) é um sinal clássico de **Reverse Shell** (ataque onde a máquina da vítima se conecta ao computador do atacante, contornando firewalls que bloqueiam conexões de entrada, mas permitem as de saída).
**Conexões Atípicas (Foreign Address)**: O processo `sh` (PID 3125) está conectado ao IP externo `45.79.12.11`. A porta local é alta (`44321`), indicando que indica que o servidor iniciou uma conexão de saída para entregar o controle ao invasor.
**Uso de Recursos (Send-Q)**: Na última linha, o Send-Q está em `256`, indicando que já dados sendo enviados mas ainda não confirmados. Em um cenário de exfiltração, esse número costuma ficar alto e constante.

##### Conclusão:
Embora a porta 80 e 22 sejam serviços padrão, o conjunto da obra (**Netcat escutando + Shell conectado para fora**) indica que o servidor está comprometido. Mesmo que fosse um “teste” de um administrador preguiçoso, seria considerado uma **má prática grave** ou uma **sombra de TI (Shadow IT).**

## 📌 Comando

### ss (socket statistics)

#### O que faz
Assim como o `netstat`, o `ss`, lista conexões de rede ativas e portas em escuta, incluindo o processo associado.
, porém é um processo mais rápido, detalhado e moderno.
Embora as flags básicas sejam idênticas, o `ss` é ótimo em cenários de alta carga porque extrai as informações diretamente de dentro do kernel (usando o subsistema *netlink*), enquanto o `netstat` precisa ler arquivos no diretório /proc, o que torna lento em servidores com milhares de conexões.

#### Uso em SOC
Funciona quase igual ao `netstat`, com algumas diferenças, sendo elas: diferença visual (o `ss` é mais limpo e não tenta “adivinhar” o cabeçalho se não houver dados); enquanto no `netstat` é preciso usar o `grep` para filtrar, o `ss` tem filtros nativos, como filtrar por estado (`ss -t state established`), filtrar por porta específica (`ss -at ‘( dport = :22 or sport = :22 )’`); filtrar por IP (`ss -nt dst 45.79.12.11`). Além disso, com a flag `-i`, o `ss` pode dar informações sobre a saúde da conexão (RTT, algoritmos de congestionamento), o que ajuda a identificar se um exfil de dados está sendo limitado pela rede ou se está “voando”.

#### Exemplo
```bash
ss -antp
ss -i
```

## 🧩 Quando eu usaria isso?

- Investigar conexões suspeitas
- Identificar comunicação com IPs externos desconhecidos
- Detectar backdoors e reverse shells
