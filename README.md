# Brute Force Attack Lab — Medusa + Kali Linux

Projeto de laboratório para o desafio da DIO: simulação de ataques de força bruta contra serviços vulneráveis (FTP, aplicação web DVWA e SMB), em ambiente controlado e isolado, com documentação das técnicas de mitigação.

> ⚠️ **Uso ético:** todos os testes foram realizados em máquinas virtuais próprias, isoladas em rede host-only, sem acesso à internet ou a sistemas de terceiros. Nunca utilize essas técnicas contra sistemas sem autorização explícita.

---

## 1. Ambiente de testes

| Papel | Sistema | IP (host-only) |
|---|---|---|
| Atacante | Kali Linux 2024.x | 192.168.56.10 |
| Alvo | Metasploitable 2 | 192.168.56.20 |
| Alvo (web) | DVWA (rodando no Metasploitable ou em outra VM) | 192.168.56.20 |

### Configuração da rede

1. No VirtualBox: **Arquivo → Preferências de Rede → Redes somente-host** → criar uma rede (ex.: `vboxnet0`), DHCP desabilitado ou com range fixo.
2. Em cada VM: **Configurações → Rede → Adaptador 2** → habilitar, modo "Rede Exclusiva do Hospedeiro (Host-Only)", selecionando a rede criada.
3. Confirmar conectividade:
   ```bash
   # No Kali
   ping 192.168.56.20
   ```
4. Mapear portas abertas no alvo:
   ```bash
   nmap -sV -p- 192.168.56.20
   ```

---

## 2. Conceitos (teoria)

- **Ataque de força bruta**: tentativa sistemática de combinações de usuário/senha até encontrar uma válida. Pode ser *dicionário* (wordlist conhecida) ou *exaustivo* (todas as combinações possíveis).
- **Password spraying**: variação que testa **poucas senhas comuns contra muitos usuários**, evitando bloqueios por tentativas excessivas em uma única conta.
- **Por que cada serviço é vulnerável:**
  - **FTP**: protocolo antigo, muitas vezes sem limite de tentativas nem MFA.
  - **Formulário web (DVWA)**: autenticação via POST sem captcha/rate limiting (nos níveis de segurança baixos do DVWA).
  - **SMB**: permite enumeração de usuários e autenticação NTLM, que pode ser testada remotamente sem bloqueio imediato.
- **Mitigações gerais**: rate limiting, bloqueio de conta após N tentativas, CAPTCHA, MFA, fail2ban/IDS, senhas fortes e únicas, desabilitar contas/serviços não usados, monitoramento de logs de autenticação.

---

## 3. Wordlists

Wordlists simples criadas para o laboratório (arquivos em `/wordlists`):

- `usuarios.txt` — usuários comuns de teste (ex.: `msfadmin`, `admin`, `user`, `root`)
- `senhas.txt` — senhas fracas comuns (ex.: `123456`, `password`, `msfadmin`, `admin123`)

> Para listas maiores, usar as wordlists do Kali em `/usr/share/wordlists/rockyou.txt` (descompactar antes: `gunzip rockyou.txt.gz`).

---

## 4. Cenário 1 — Força bruta em FTP com Medusa

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp
```

- `-h` host alvo
- `-U` / `-P` wordlists de usuário/senha
- `-M ftp` módulo do serviço

**Validação:** ao encontrar uma combinação válida, o Medusa exibe `ACCOUNT FOUND`. Confirmar manualmente:
```bash
ftp 192.168.56.20
```

**Evidência:** capturar print do terminal com o resultado (salvar em `/images/ftp-medusa.png`).

---

## 5. Cenário 2 — Automação em formulário web (DVWA)

Com o DVWA no nível de segurança **low**, a página de login aceita tentativas sem limitação.

1. Identificar os parâmetros do formulário de login (usuário, senha, token CSRF se houver) com o navegador (F12 → Network) ou Burp Suite.
2. Usar o módulo `http` do Medusa (ou Hydra como alternativa) apontando para o formulário:
   ```bash
   medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt \
     -M web-form -m FORM:"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed"
   ```
   *(sintaxe do módulo `web-form` pode variar entre versões do Medusa — documentar a versão usada e, se necessário, usar Hydra `http-post-form` como alternativa testada.)*

**Validação:** login bem-sucedido no DVWA com a credencial encontrada.

---

## 6. Cenário 3 — Password spraying em SMB com enumeração de usuários

1. Enumerar usuários e compartilhamentos SMB:
   ```bash
   enum4linux -a 192.168.56.20
   # ou
   smbclient -L //192.168.56.20 -N
   ```
2. Fazer password spraying (poucas senhas comuns contra os usuários enumerados):
   ```bash
   medusa -h 192.168.56.20 -U usuarios_enumerados.txt -P senhas_comuns.txt -M smbnt
   ```

**Validação:** acesso autenticado a um compartilhamento:
```bash
smbclient //192.168.56.20/share -U usuario%senha
```

---

## 7. Resultados e evidências

| Cenário | Serviço | Credencial encontrada | Tempo/tentativas | Evidência |
|---|---|---|---|---|
| 1 | FTP | *(preencher)* | *(preencher)* | `/images/ftp-medusa.png` |
| 2 | DVWA (web) | *(preencher)* | *(preencher)* | `/images/dvwa-login.png` |
| 3 | SMB | *(preencher)* | *(preencher)* | `/images/smb-access.png` |

---

## 8. Recomendações de mitigação

- Implementar **bloqueio de conta** após N tentativas falhas (ex.: 3–5) em FTP, SMB e no formulário web.
- Adicionar **CAPTCHA** e token CSRF ao formulário de login do DVWA (ou de qualquer aplicação real).
- Usar **fail2ban** para banir IPs com tentativas repetidas de login em FTP/SSH/SMB.
- Desabilitar contas e serviços legados não utilizados (ex.: FTP anônimo, SMBv1).
- Aplicar **MFA** sempre que possível, especialmente em serviços expostos.
- Monitorar logs de autenticação com um SIEM ou ferramentas simples (ex.: `journalctl`, `auditd`).
- Adotar política de senhas fortes e exclusivas, com gerenciador de senhas.

---

## 9. Referências

- [Kali Linux](https://www.kali.org/)
- [DVWA](http://www.dvwa.co.uk/)
- [Medusa](http://www.foofus.net/jmk/medusa/medusa.html)
- [Nmap](https://nmap.org/book/)

---

## Estrutura do repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   └── senhas.txt
└── images/
    ├── ftp-medusa.png
    ├── dvwa-login.png
    └── smb-access.png
```
