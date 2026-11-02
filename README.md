# Simulando ataques de senhas com Medusa e Kali Linux

Projeto prático da Aceleração Santander — Cibersegurança do Zero à Prática, da DIO.

Autor: Alexandro Bastos do Nascimento

## Objetivo

Estudar ataques de dicionário e password spraying em serviços FTP, formulário web e SMB, registrando os resultados e validando as credenciais encontradas.

Todos os testes foram realizados em máquinas virtuais próprias, em um laboratório com rede Host-only.

## Ambiente

- Virtualização: VirtualBox.
- Kali Linux: 192.168.56.102/24.
- Metasploitable 2: 192.168.56.101/24.
- Ferramentas: Nmap, Medusa 2.3, Hydra 9.6, enum4linux, cliente FTP e smbclient.

## Reconhecimento

Foi executado o comando:

```bash
nmap -sV -p 21,22,80,139,445 -oN servicos-lab.txt 192.168.56.101
```

Foram identificados os serviços:

- FTP: vsftpd 2.3.4, porta 21.
- SSH: OpenSSH 4.7p1, porta 22.
- HTTP: Apache 2.2.8, porta 80.
- SMB: Samba, portas 139 e 445.

Registro: [servicos-lab.txt](resultados/servicos-lab.txt).

## Listas utilizadas

O arquivo users.txt contém:

```text
user
msfadmin
admin
root
```

O arquivo pass.txt contém:

```text
123456
password
qwerty
msfadmin
```

As listas pequenas permitiram observar o comportamento das ferramentas sem executar um volume elevado de tentativas.

## 1. Teste FTP com Medusa

Comando executado:

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 1 -O resultados/medusa-ftp.txt
```

Resultado: o Medusa identificou a credencial msfadmin/msfadmin.

A validação foi realizada com:

```bash
ftp 192.168.56.101
```

Após informar as credenciais, o servidor respondeu com “230 Login successful”. A sessão foi encerrada com bye, sem transferência de arquivos.

Na primeira execução, foi usado -u users.txt, que interpreta users.txt como um único nome de usuário. A opção foi corrigida para -U users.txt, que lê a lista.

Registro: [medusa-ftp.txt](resultados/medusa-ftp.txt).

## 2. Teste do formulário DVWA

O formulário foi analisado pelas ferramentas de desenvolvedor do navegador, na aba Network.

Foram identificados:

- Método: POST.
- Caminho: /dvwa/login.php.
- Campos: username, password e Login.
- Mensagem de falha: Login failed.
- Redirecionamento HTTP 302 após o envio.

### Limitação encontrada no Medusa

O módulo web-form foi consultado e testado. Nessa execução, o Medusa retornou erro ao interpretar a resposta HTTP 302.

Esse erro não foi tratado como autenticação bem-sucedida. Foi utilizado o Hydra para continuar o teste do formulário.

Ajuda consultada: [ajuda-web-form.txt](resultados/ajuda-web-form.txt).

### Validação da configuração do Hydra

Antes de usar as listas, foram realizados dois controles:

- teste/senhaerrada: zero senhas válidas.
- admin/password: uma senha válida.

A credencial admin/password também foi confirmada manualmente no navegador, com acesso à página inicial do DVWA.

### Teste com listas

Os parâmetros foram preparados no mesmo terminal:

```bash
alvo=192.168.56.101
pagina=/dvwa/login.php
dados='username=^USER^&password=^PASS^&Login=Login'
falha='F=Login failed'
saida=resultados/hydra-dvwa.txt
```

Comando executado:

```bash
hydra -L users.txt -P pass.txt -t 1 -o "$saida" "$alvo" http-post-form "$pagina:$dados:$falha"
```

Resultado: o teste com as listas encontrou admin/password.

Uma execução anterior retornou zero resultados porque o nome do campo username havia sido digitado incorretamente. Após a correção, a credencial foi identificada. O registro preserva as duas execuções.

Registro: [hydra-dvwa.txt](resultados/hydra-dvwa.txt).

## 3. Enumeração e password spraying SMB

A enumeração foi realizada com:

```bash
enum4linux -a 192.168.56.101 > resultados/enum4linux.txt 2>&1
```

A saída revelou usuários e grupos do servidor. Foram selecionados três usuários identificados para o arquivo smb_users.txt:

```text
user
msfadmin
service
```

Registro: [enum4linux.txt](resultados/enum4linux.txt).

### Password spraying

Foi testada uma única senha, msfadmin, nos três usuários:

```bash
medusa -h 192.168.56.101 -U smb_users.txt -p msfadmin -M smbnt -t 1 -O resultados/medusa-smb.txt
```

Resultado: o Medusa identificou msfadmin/msfadmin e informou acesso ao compartilhamento ADMIN$.

O teste caracteriza password spraying porque aplicou a mesma senha a diferentes contas.

Durante a execução original, o nome do arquivo de saída foi digitado como %registro. O arquivo foi posteriormente renomeado e movido para resultados/medusa-smb.txt.

Registro: [medusa-smb.txt](resultados/medusa-smb.txt).

### Validação com smbclient

Foi executado:

```bash
smbclient -L //192.168.56.101 -U msfadmin
```

Após informar a senha msfadmin, foram listados os compartilhamentos print$, tmp, opt, IPC$, ADMIN$ e msfadmin.

Essa validação confirmou a listagem dos compartilhamentos. Não foram realizados testes de leitura, escrita ou execução de arquivos.

Registro: [smb-validacao.txt](resultados/smb-validacao.txt).

## Resultados

- FTP: msfadmin/msfadmin, identificado pelo Medusa e validado pelo cliente FTP.
- DVWA: admin/password, identificado pelo Hydra e validado no navegador.
- SMB: msfadmin/msfadmin, identificado pelo Medusa e validado com smbclient.

As credenciais apresentadas pertencem exclusivamente ao laboratório vulnerável utilizado neste projeto.

## Medidas de proteção

- Substituir credenciais padrão e evitar senhas previsíveis ou reutilizadas.
- Adotar senhas longas e exclusivas e autenticação multifator quando disponível.
- Aplicar limitação de tentativas e atrasos progressivos, considerando também ataques distribuídos entre contas.
- Monitorar falhas e sucessos de autenticação e gerar alertas para padrões suspeitos.
- Restringir a exposição dos serviços por firewall e segmentação de rede.
- Substituir FTP sem criptografia por alternativas como SFTP.
- Atualizar os serviços e desabilitar protocolos antigos, como SMB1.
- Restringir enumeração anônima e permissões dos compartilhamentos.

## Aprendizados

O projeto mostrou que uma mensagem de sucesso de uma ferramenta precisa ser validada e que erros de conexão não equivalem à rejeição de uma senha.

Também demonstrou a importância de conferir os nomes dos campos do formulário, distinguir opções maiúsculas e minúsculas e preservar os registros das correções realizadas.

As variáveis do terminal precisam ser definidas novamente ao abrir uma nova sessão.

## Evidências

Os registros em texto estão na pasta resultados. O arquivo rede-kali.txt documenta as interfaces e o endereço do Kali.

As capturas de tela foram utilizadas durante o acompanhamento dos testes. Os arquivos de texto permitem consultar os resultados com maior legibilidade.
