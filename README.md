 # Auditoria de senhas em laboratório: Kali Linux + Medusa

> Projeto educacional para demonstrar auditoria de autenticação em ambiente **próprio, isolado e autorizado**. Os comandos abaixo devem ser executados somente contra as VMs do laboratório.

## 1. Objetivo

Montar um laboratório com Kali Linux e Metasploitable 2 no VirtualBox e estudar, com listas pequenas, como tentativas de autenticação podem afetar FTP, SMB e um formulário web do DVWA. Ao final, registrar evidências, validar resultados e propor controles defensivos.

## 2. Escopo e segurança

- Alvos permitidos: somente as VMs criadas pelo próprio aluno.
- Rede: Host-only (ou rede interna isolada), sem encaminhamento de portas para a internet.
- Não reutilizar estas listas/comandos em sistemas de terceiros.
- Limitar tentativas e paralelismo; interromper o teste se houver instabilidade.
- Não publicar senhas reais, cookies, tokens, endereços públicos ou dados pessoais nas evidências.

## 3. Ambiente

| Máquina | Função | Configuração sugerida |
|---|---|---|
| Kali Linux | Estação de auditoria | 2 vCPU, 2–4 GB RAM |
| Metasploitable 2 | Alvo intencionalmente vulnerável | 1 vCPU, 1 GB RAM |

No VirtualBox, configure **as duas VMs na mesma rede Host-only** (por exemplo, `vboxnet0`). Não use modo Bridge. Anote os IPs obtidos:

- IP do Kali: `PREENCHER`
- IP do Metasploitable 2: `PREENCHER`

No Kali, descubra o IP e confirme a conectividade:

```bash
ip -br a
ping -c 3 IP_DO_METASPLOITABLE
```

No alvo, confirme os serviços disponíveis a partir do Kali:

```bash
nmap -sV -p 21,80,139,445 IP_DO_METASPLOITABLE
```

Salve a saída em `evidencias/nmap-servicos.txt`. Não presuma que uma porta está aberta: use o resultado real do seu laboratório.

## 4. Preparação do Kali

```bash
sudo apt update
sudo apt install -y medusa nmap ftp smbclient
medusa -d
```

O último comando lista os módulos instalados. Para consultar as opções de um módulo:

```bash
medusa -M ftp -q
medusa -M smbnt -q
medusa -M web-form -q
```

Se algum módulo não aparecer em `medusa -d`, registre a limitação e não invente uma saída.

## 5. Wordlists pequenas

As listas deste repositório são exemplos didáticos, não listas para uso em sistemas reais. Confira o conteúdo antes de cada execução:

```bash
cat wordlists/usuarios-lab.txt
cat wordlists/senhas-lab.txt
```

Os arquivos contêm somente contas/senhas de laboratório comuns. Adapte-os apenas a contas criadas intencionalmente no seu alvo.

## 6. Cenário A — auditoria de FTP com Medusa

Verifique primeiro se a porta 21 está aberta no alvo. Depois, execute um teste pequeno, com baixa concorrência:

```bash
medusa -h IP_DO_METASPLOITABLE \
  -U wordlists/usuarios-lab.txt \
  -P wordlists/senhas-lab.txt \
  -M ftp -t 1 -T 1 \
  -O evidencias/medusa-ftp.txt
```

Valide qualquer credencial indicada como sucesso de forma independente, usando o cliente FTP:

```bash
ftp IP_DO_METASPLOITABLE
```

Registre se a autenticação foi confirmada ou se houve falso positivo. Não copie senhas válidas para o relatório público; informe apenas o usuário de laboratório e o resultado, ou mascare a credencial.

## 7. Cenário B — password spraying em SMB

Password spraying testa uma senha de cada vez contra vários usuários. Use **uma única senha de laboratório por execução**, uma lista curta de contas de teste e baixa concorrência:

```bash
medusa -h IP_DO_METASPLOITABLE \
  -U wordlists/usuarios-lab.txt \
  -p SENHA_DE_TESTE_DO_LAB \
  -M smbnt -t 1 -T 1 \
  -O evidencias/medusa-smb-spray.txt
```

Consulte o módulo local (`medusa -M smbnt -q`) antes de executar. A disponibilidade e o comportamento do SMB variam conforme a versão/configuração do alvo. Se não houver usuários de teste conhecidos, não faça enumeração invasiva: registre a limitação e use contas criadas por você no laboratório.

Valide o resultado com `smbclient` somente se o serviço estiver ativo e a conta pertencer ao seu laboratório:

```bash
smbclient -L //IP_DO_METASPLOITABLE -U USUARIO_DE_TESTE
```

## 8. Cenário C — formulário Brute Force do DVWA

O DVWA possui um módulo próprio em **Vulnerabilities → Brute Force**. Ele é diferente da tela de autenticação inicial do DVWA. Faça login na aplicação, configure o nível de segurança conforme a atividade (por exemplo, Low, apenas no laboratório) e identifique:

- URL do módulo: `http://IP_DO_ALVO/dvwa/vulnerabilities/brute/` (ajuste ao seu caminho real);
- método HTTP (GET/POST);
- nomes dos campos de usuário e senha;
- mensagem que aparece quando a tentativa falha;
- presença de cookie de sessão ou token anti-CSRF.

O Medusa oferece o módulo `web-form`, mas a compatibilidade depende dos parâmetros do formulário e da versão do DVWA/Medusa. Consulte:

```bash
medusa -M web-form -q
```

Modelo **a ser ajustado após inspecionar o formulário** (não execute sem confirmar os campos e a mensagem de falha):

```bash
medusa -h IP_DO_ALVO -u USUARIO_DE_TESTE \
  -P wordlists/senhas-lab.txt \
  -M web-form \
  -m FORM:"/dvwa/vulnerabilities/brute/" \
  -m FORM-DATA:"get?username=&password=&Login=Login" \
  -m DENY-SIGNAL:"TEXTO_EXATO_DE_FALHA" \
  -t 1 -T 1 \
  -O evidencias/medusa-dvwa.txt
```

**Atenção à interpretação:** o DVWA pode exigir uma sessão autenticada, cookie e/ou token CSRF. Além disso, o módulo `web-form` identifica falhas por uma string configurada; se a resposta não for distinguível, ele pode reportar falso positivo. Não declare uma senha descoberta sem validar manualmente no navegador e confirmar que o login realmente ocorreu. Se o Medusa não conseguir tratar a sessão/token da versão instalada, documente a limitação — isso é um resultado técnico válido. Não apresente uma execução de outra ferramenta como se fosse do Medusa.

## 9. Resultados e evidências

Preencha a tabela somente depois de executar os testes. Não invente IPs, resultados, capturas ou credenciais.

| Cenário | Serviço/módulo | Tentativas | Resultado observado | Validação independente |
|---|---|---:|---|---|
| FTP | FTP / Medusa | `PREENCHER` | `PREENCHER` | `PREENCHER` |
| SMB spraying | SMBNT / Medusa | `PREENCHER` | `PREENCHER` | `PREENCHER` |
| DVWA | web-form / Medusa | `PREENCHER` | `PREENCHER` | `PREENCHER` |

Adicione capturas próprias em `evidencias/`:
1. Configuração Host-only das duas VMs (sem dados sensíveis).
2. Resultado do Nmap mostrando apenas os serviços do laboratório.
3. Execução do Medusa para FTP.
4. Execução do password spraying SMB.
5. Tela do DVWA e evidência da validação manual.
6. Se houver falha/limitação, captura da mensagem de erro e explicação.

## 10. Análise e mitigação

| Risco observado | Medida recomendada |
|---|---|
| Senhas fracas ou reutilizadas | Política de senhas robustas, bloqueio de senhas conhecidas/vazadas e MFA |
| Muitas tentativas em curto período | Rate limiting, atraso progressivo e bloqueio temporário baseado em risco |
| Password spraying distribuído | Monitoramento de falhas por origem e por conta; correlação no SIEM |
| Contas padrão/compartilhadas | Remover contas padrão, individualizar contas e aplicar menor privilégio |
| SMB exposto desnecessariamente | Restringir TCP 445 por firewall/VLAN; desabilitar SMBv1 e exigir SMB signing quando aplicável |
| FTP sem criptografia | Substituir por SFTP/FTPS; desabilitar FTP legado quando possível |
| Formulário web sem proteção | MFA, limitação de tentativas, mensagens genéricas, logs, proteção de sessão e tokens CSRF adequados |

## 11. Conclusão — modelo para completar

> Neste laboratório, configurei duas máquinas virtuais isoladas para estudar tentativas de autenticação em serviços de rede e aplicação web. Utilizei Nmap para identificar os serviços expostos e Medusa para testar, de forma controlada, os módulos FTP e SMBNT. No DVWA, avaliei o formulário Brute Force e a compatibilidade do módulo web-form, considerando sessão, tokens e detecção de falhas. Os testes demonstraram que resultados de ferramentas precisam ser validados independentemente. Como medidas defensivas, recomendo MFA, limitação de tentativas, senhas fortes, monitoramento centralizado e redução da superfície de exposição.

Edite este parágrafo para refletir **o que realmente aconteceu** no seu ambiente.

## 12. Referências

- [Medusa — documentação do projeto](https://jmk-foofus.github.io/medusa/medusa.html)
- [Medusa — manual (Debian)](https://sources.debian.org/src/medusa/2.1.1-1/doc/medusa.1)
- [Kali Linux — documentação oficial](https://www.kali.org/docs/)
- [DVWA — projeto](https://github.com/digininja/DVWA)
- [Nmap — referência oficial](https://nmap.org/book/man.html)
- [GitHub — guia de início rápido](https://docs.github.com/pt/get-started)

