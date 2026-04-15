## 🎯 Objetivo da análise

Nesta etapa, buscamos identificar comportamentos anormais relacionados a processos em execução no sistema.

# 🧠 Análise de Processos

## 📌 Comandos

### ps aux

#### O que faz
Lista todos os processos ativos no sistema.

#### Uso em SOC
Identificar processos suspeitos, malware ou execução não autorizada.

#### Exemplo
```bash
ps aux
```

O que observar
Processos desconhecidos
Execução como root
Alto uso de CPU/memória

## 🧪 Mini cenário

Durante uma investigação, foi identificado alto uso de CPU.

### Passos:
1. Executar `ps aux`
2. Identificar processo suspeito
3. Investigar PID
