# Privilege Escalation Risk Analyzer - Linux Sandbox Lab

## Introdução
Este projeto tem como objetivo criar um ambiente sandbox em Linux para análise de inconsistências de processos, simular isolamento usando `bubblewrap` (`bwrap`) e explorar técnicas de visualização de PID, visando o estudo de segurança, contêineres e escalonamento de privilégios.

O laboratório consiste em: 
- Isolamento do sistema de arquivos e PIDs
- Comparação entre processos visíveis via `/proc` e `ps`
- Identificação de inconsistências ou processos escondidos

## Pré-requisitos
- Sistema Linux com `bash`
- `sudo` com permissões de root
- Ferramentas: `bubblewrap (bwrap)`, `ps`, `ls`, `mount`, `chroot`
- Acesso à pasta de trabalho (ex: `/home/marcel/security-lab`)

## Estrutura de diretórios usada
```
/home/marcel/security-lab/
└── chroot/
    ├── bin
    ├── dev
    ├── home
    ├── lib
    ├── lib64
    ├── media
    ├── mnt
    ├── opt
    ├── proc
    ├── root
    ├── run
    ├── sbin
    ├── srv
    ├── sys
    ├── tmp
    ├── usr
    └── var
```

## Comandos e Procedimentos Executados

### 1. Preparação do sandbox com `bwrap`
```bash
# Verificando instalação do bwrap
which bwrap

# Criando sandbox com binds de diretórios essenciais e tmpfs
bwrap \
  --ro-bind /usr /usr \
  --ro-bind /bin /bin \
  --ro-bind /lib /lib \
  --ro-bind /lib64 /lib64 \
  --proc /proc \
  --dev /dev \
  --tmpfs /tmp \
  /bin/bash

# Validando isolamento
echo "PID:" $$  # PID dentro do sandbox
ls /
touch /test-host
ls /tmp
```

### 2. Uso avançado de isolamento e unshare
```bash
sudo unshare --pid --fork --mount --uts --ipc --mount-proc bash
mount -t proc proc /proc
mount --rbind /sys /sys
mount --rbind /dev /dev
chroot /home/marcel/security-lab/chroot /bin/bash
```
- Dentro do chroot, o PID inicial era diferente (`echo $$`)
- Validado que processos do host não são visíveis dentro do sandbox

### 3. Coleta de baseline de processos e rootfs
```bash
mkdir -p /tmp/lab
cd /tmp/lab
ps aux > baseline_ps.txt
ls -la / > baseline_rootfs.txt
```
- Arquivos `baseline_ps.txt` e `baseline_rootfs.txt` armazenam o estado inicial para comparação

### 4. Identificação de inconsistências entre `/proc` e `ps`
```bash
ls /proc | grep -E '^[0-9]+$' > proc_pids.txt
ps -eo pid > ps_pids.txt
```
- Comparando diferenças:
```bash
cat << 'EOF' > detect_inconsistency.sh
#!/bin/bash

echo "[+] PIDs via /proc:"
ls /proc | grep -E '^[0-9]+$' | sort > /tmp/proc_view.txt
cat /tmp/proc_view.txt

echo
 echo "[+] PIDs via ps:"
ps -eo pid | tr -d ' ' | tail -n +2 | sort > /tmp/ps_view.txt
cat /tmp/ps_view.txt

echo
 echo "[!] Diferença (/proc - ps):"
comm -23 /tmp/proc_view.txt /tmp/ps_view.txt
EOF
chmod +x detect_inconsistency.sh
./detect_inconsistency.sh
```
- Diferenças indicam processos que aparecem no `/proc` mas não em `ps`, simulando processos ocultos

### 5. Limpeza do ambiente
```bash
exit  # sair do chroot
sudo umount -l /home/marcel/security-lab/chroot/proc
sudo umount -l /home/marcel/security-lab/chroot/sys
sudo umount -l /home/marcel/security-lab/chroot/dev
```

## Scripts Criados
- `detect_inconsistency.sh` → detecta diferenças entre a lista de PIDs do `/proc` e do `ps`

## Testes e Observações
- Verificação do PID dentro e fora do sandbox (`echo $$`) → valida isolamento
- `ls /home` não acessível dentro do sandbox → confirma isolamento do FS
- Arquivos criados no `/tmp` são temporários e limpos ao reiniciar sandbox
- Processos invisíveis via `ps` mas visíveis via `/proc` → teste de inconsistência

### Erros enfrentados
- `chroot: não foi possível alterar o diretório raiz` → corrigido garantindo existência da pasta
- Comando digitado `exist` em vez de `exit` → erro de digitação

## Ferramentas Utilizadas
- `bubblewrap (bwrap)` → criação de sandbox isolado
- `unshare`, `chroot` → isolamento de PIDs e FS
- `ls`, `ps`, `mount` → monitoramento de estado
- Scripts Bash → automação de validação

## Próximos passos
- Automatizar coleta e comparação de processos em múltiplos sandboxes
- Criar alertas para inconsistências detectadas
- Explorar integração com auditoria de segurança ou containers Docker

---

*Este projeto é voltado para estudos de segurança e auditoria de processos no Linux.*
