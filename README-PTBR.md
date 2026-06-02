# TryHackMe - Wgel CTF Write-up

## Introdução

O objetivo desta sala foi obter acesso inicial à máquina alvo por meio de enumeração web e, em seguida, realizar uma escalada de privilégios para obter a flag de root.

Este desafio aborda técnicas de reconhecimento, enumeração de diretórios, autenticação SSH utilizando uma chave privada exposta e privilege escalation através de permissões sudo mal configuradas.

---

# Reconhecimento

O primeiro passo foi realizar um scan com o Nmap para identificar portas abertas e serviços em execução.

```bash
nmap 10.66.XXX.XXX -sS -sV -sC
```

<br>

<img width="804" height="310" alt="image" src="https://github.com/user-attachments/assets/19407278-58b1-41c1-8ef8-26723b22df91" />

<br><br>

Após analisar os resultados, identifiquei dois serviços interessantes:

* SSH rodando na porta 22
* Servidor Apache HTTP rodando na porta 80

Com essas informações, decidi iniciar a investigação pela aplicação web.

---

# Enumeração Web

Ao acessar o servidor web, encontrei a página padrão do Apache2 no Ubuntu.

<br>

<img width="844" height="817" alt="image" src="https://github.com/user-attachments/assets/5d0b6cc5-d95a-4557-85a7-dc92776c0e68" />

<br><br>

Como páginas padrão frequentemente escondem conteúdos adicionais, realizei uma enumeração de diretórios no servidor web.

<br>

<img width="995" height="407" alt="image" src="https://github.com/user-attachments/assets/636d7806-277e-4d7a-a517-7c22cb0d23bd" />

<br><br>

Durante a enumeração, descobri o diretório:

```text
/sitemap
```

Ao acessar esse diretório, encontrei uma aplicação web contendo diversas páginas e seções de navegação.

Antes de continuar, explorei manualmente o site e coletei informações que poderiam ser úteis posteriormente, como nomes de funcionários, endereços de e-mail e outros dados expostos.

Mesmo que essas informações não tenham sido utilizadas diretamente na exploração, documentar tudo o que for encontrado durante a fase de reconhecimento é uma boa prática.

Após a análise manual do site, realizei uma nova enumeração de diretórios, desta vez focada especificamente no diretório `/sitemap`.

<br>

<img width="1485" height="769" alt="image" src="https://github.com/user-attachments/assets/10fbb633-fb42-46ea-b9e6-05f6ac304771" />

<br><br>

Essa enumeração revelou um diretório bastante interessante:

```text
/.ssh
```

<br>

<img width="995" height="482" alt="image" src="https://github.com/user-attachments/assets/84433da5-7c1d-4f88-b243-00dcd23c8f5c" />

<br><br>

Dentro dele encontrei uma chave privada SSH exposta:

```text
id_rsa
```

<br>

<img width="1846" height="687" alt="image" src="https://github.com/user-attachments/assets/0a557545-d2e9-475c-b4ea-36a8326e49a9" />

<br><br>

Ao abrir o arquivo, foi possível visualizar uma chave RSA completa.

<br>

<img width="707" height="560" alt="image" src="https://github.com/user-attachments/assets/eae4c1a8-f7b2-45f0-b09e-382eb2cea2a1" />

<br><br>

---

# Acesso Inicial

Para utilizar a chave descoberta, criei um arquivo local contendo seu conteúdo.

<br>

<img width="265" height="56" alt="image" src="https://github.com/user-attachments/assets/e20bcbc1-8851-4d8e-9497-0cea68caf132" />

<br><br>

Em seguida, ajustei suas permissões para que o SSH aceitasse a chave.

```bash
chmod 600 id_rsa
```

<br>

<img width="277" height="60" alt="image" src="https://github.com/user-attachments/assets/a118d1dc-f2b5-4c1d-ae16-14719b171ce2" />

<br><br>

Neste momento eu ainda precisava descobrir qual era o usuário válido para autenticação.

Inicialmente tentei utilizar nomes de desenvolvedores encontrados durante a enumeração do site, mas sem sucesso.

Então voltei a analisar o código-fonte das páginas e, ao inspecionar a página padrão do Apache, encontrei um comentário contendo o nome do usuário:

```text
jessie
```

<br>

<img width="618" height="215" alt="image" src="https://github.com/user-attachments/assets/9a07213b-64a9-4366-b75a-1f8d7cfa78d3" />

<br><br>

Com a chave privada e o nome do usuário em mãos, consegui realizar a autenticação via SSH.

```bash
ssh -i id_rsa jessie@TARGET_IP
```

<br>

<img width="546" height="221" alt="image" src="https://github.com/user-attachments/assets/e89e4d85-3acb-48d5-bff2-c761a96442a4" />

<br><br>

Após obter acesso ao sistema, explorei os diretórios do usuário e encontrei a primeira flag.

```text
user_flag.txt
```

<br>

<img width="742" height="131" alt="image" src="https://github.com/user-attachments/assets/6632a351-60e0-449e-acfe-51eba348cea3" />

<br><br>

---

# Escalada de Privilégios

A sala também exige a obtenção da flag de root.

Para identificar possíveis vetores de privilege escalation, executei:

```bash
sudo -l
```

<br>

<img width="958" height="118" alt="image" src="https://github.com/user-attachments/assets/6e8d21d7-0401-4839-a94a-7407aff5ffc4" />

<br><br>

O resultado revelou a seguinte permissão:

```text
(root) NOPASSWD: /usr/bin/wget
```

Isso significa que o usuário pode executar o binário `wget` como root sem a necessidade de fornecer senha.

Para entender como essa permissão poderia ser explorada, consultei o GTFOBins e encontrei uma técnica que permite utilizar o `wget` para enviar o conteúdo de arquivos privilegiados para uma máquina controlada pelo atacante.

<br>

<img width="864" height="248" alt="image" src="https://github.com/user-attachments/assets/30be51ac-5f1c-4c3f-830c-9a9256d5667f" />

<br><br>

Primeiramente configurei um listener na máquina atacante:

```bash
nc -lvnp 4444
```

Em seguida, utilizei o `wget` na máquina alvo para enviar o conteúdo do arquivo da flag de root para o listener.

<br>

<img width="766" height="79" alt="image" src="https://github.com/user-attachments/assets/e1238a73-97cf-4a0c-8ed2-7f5ad5d312ba" />

<br><br>

O listener recebeu com sucesso o conteúdo do arquivo.

<br>

<img width="739" height="250" alt="image" src="https://github.com/user-attachments/assets/fbe14b33-44eb-414e-8005-c1a43e255f4b" />

<br><br>

Com isso, foi possível obter a flag de root e concluir o desafio.

---

# Conclusão

Esta sala demonstra muito bem como arquivos sensíveis expostos em um servidor web podem resultar no comprometimento completo de um sistema.

A cadeia de ataque foi composta pelas seguintes etapas:

1. Enumeração do servidor web.
2. Descoberta do diretório `/sitemap`.
3. Localização de uma chave privada SSH exposta.
4. Identificação do usuário válido através da análise do código-fonte.
5. Obtenção de acesso via SSH.
6. Exploração de uma permissão sudo insegura envolvendo o `wget`.
7. Recuperação da flag de root.

O desafio reforça a importância de proteger arquivos sensíveis, restringir diretórios acessíveis pela web e revisar cuidadosamente permissões sudo concedidas aos usuários.
