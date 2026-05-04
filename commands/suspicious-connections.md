## 🎯 Objetivo da análise

O `lsof` significa **”List Open Files”**. No Linux, como quase tudo é tratado como arquivo, as conexões de rede também são listadas aqui. A flag `-i` especificamente filtra para mostrar apenas arquivos de internet (sockets de rede).

# 🌐 Análise de Conexões Suspeitas

## 📌 Comando

### lsof -i

#### O que ele faz
O comando lista todos os arquivos abertos por todos os processos ativos. Com a flag `-i`, ele revela quais processos possuem conexões TCP ou UDP abertas, mostrando o endereço local, o endereço remoto e o estado da conexão.

#### Uso em SOC
Identificar exatamente qual executável (caminho completo) está segurando aquela conexão; investigar processos sem nome ( `lsof -i` dificulta que conexões de rede passem despercebidas); e verificar escuta (LISTEN) não autorizada.

#### Exemplo
```bash
lsof -i
```

### O que cada flag faz

##### `-i`: Lista todas as conexões de rede.
##### `-i :80`: Lista apenas quem está usando a porta 80.
##### `-i tcp`: Filtra apenas por conexões TCP.
##### `-i @[REDACTED]`: Mostra conexões relacionadas a um IP específico.
### Flags Úteis
##### `-n`: Não resolve nomes de host (mostra o IP), o que é mais rápido e seguro no SOC.
##### `-P`: Não resolve nomes de portas (mostra 80 em vez de `http`).
##### `-p [PID]`: Mostra todos os arquivos e conexões de um processo específico.
## 🧪 Mini cenário: O Socket Fantasma
Foi identificado um tráfego estranho saindo do meu servidor de arquivos para um IP [REDACTED]. O `netstat` mostrou a conexão, mas preciso de mais detalhes para o relatório.

### Passos:
1. Executar `sudo lsof -i -nP`
2. Identificar processo suspeito
3. Investigar PID

#### Output simulado de processo suspeito
```bash
# sudo lsof -i -nP
COMMAND    PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      1021     root    3u  IPv4  20412      0t0  TCP *:22 (LISTEN)
[REDACTED] 8892 www-data   4u  IPv4  88921      0t0  TCP 192.168.1.10:55432->[REDACTED]:443 (ESTABLISHED)            

```

#### Investigação de PID
1. **COMMAND**: O comando aparece como `[REDACTED]` ou um nome aleatório. O `lsof` te dá o PID 8892.

2. **FD (File Descriptor)**: O valor `4u` indica que o quarto descritor de arquivo deste processo é um socket de rede (`u` significa leitura e escrita).

3. **NAME**: Aqui vemos o fluxo total. O meu servidor (`192.168.1.10`) abriu uma conexão de uma porta alta para a porta `443` (HTTPS) do IP [REDACTED].


##### Conclusão:
Embora a porta `443` pareça tráfego web comum, o processo que a originou não é um navegador nem um serviço de atualização. É um processo rodando sob o usuário `www-data`, sugerindo que o seu servidor web foi usado para enviar dados para fora (exfiltração disfarçada de HTTPS).

##### OBS sobre o cenário:
- Foi usado o comando `lsof` com as flags `-i` e `-nP`. 
Pesquisando sobre a flag `-nP`, concluo que é a junção das flags `-n` e `-p`, porém com o “P” maiúsculo após o “n” minúsculo.

Junções como essa são comuns para trazer agilidade para o comando. Nesse, em específico, estaria solicitando apenas os números brutos (mais rápido).

Sempre que estiver investigando conexões suspeitas, usar `-nP`. É o padrão da indústria para garantir que você estou vendo dados puros e não interpretados.

- Por que não pode juntar com a flag `-i`?

De acordo com as minhas pesquisas, essa é uma questão que entra uma regra de “gramática” no terminal:

Flags que **não** precisam de um complemento (como o `-n` e o `-P`) podem ser agrupadas.

Flags que **exigem** um parâmetro ou que alteram o comportamento das seguintes (como o `-i`, que muitas vezes espera que você diga a porta ou o protocolo logo depois) geralmente ficam melhor sozinhas no início ou no fim para evitar erros de interpretação no comando.

- **O que funciona:**
  
`lsof -i -nP` (A forma mais limpa e comum).

`lsof -niP` (Algumas versões do `lsof` aceitam, mas pode ser confuso).

- **O que não funciona ou é arriscado:**

`lsof -nP i` (O sistema vai achar que `i` é um arquivo que você quer listar, não a flag de internet).
