---
title: "/PROC – HidePID – Esconder Processos de outros usuários"
author: "Rafael Grether"
description: "Aprenda a esconder processos em execução de outros usuários e se proteger contra data leaks."
tags: ["seguranca", "defesa_interna"]
date: "2024-01-28"
---
Conceito, Exemplos e Formas de Proteção

<!--more-->

## Conceito

/PROC pode ser considerado como um sistema de arquivos “virtual” que contém arquivos especiais com informações do hardware, informações e configurações do sistema a um nível do kernel (como /proc/meminfo , /proc/mounts , /proc/sys) e processos em execução no sistema.

Existe um conceito que diz “everything is a file” (ou seja, tudo é um arquivo). Nisso se resume a arquitetura do sistema operacional.  
Isso significa que tudo o que existe no sistema, desde arquivos, diretórios, até sockets e processos; é um arquivo. Mais especificamente falando, um descritor de arquivo, que de modo grosseiro seria uma espécie de “código” que o sistema operacional utiliza para gerenciamento, e isso ocorre através de uma camada especial no kernel do sistema, que é o chamado sistema de arquivos “virtual”, ou PROCFS, mais conhecido como /PROC filesystem.

### Exemplo

Em um terminal, vamos criar um arquivo teste.txt e escrever algumas coisas nele:

```bash
vim teste.txt
```

Em outro terminal, ao analisarmos os processos em execução no sistema, encontramos o nosso processo criado e seu respectivo PID:

![.](https://kali.seg.br/post/imgs/hidepid_vim.webp)

Aqui, o PID 2143 que está em execução, é um descritor de arquivo que está localizado no sistema de arquivos “virtual” /PROC.  
Podemos verificar “na unha” os detalhes da execução dele diretamente pelo diretório /PROC (abaixo), no qual podemos saber o que o PID em questão está fazendo:

![.](https://kali.seg.br/post/imgs/hidepid_proc.webp)

## Problema

O problema é que em toda instalação padrão do sistema, o sistema de arquivos “virtual” /PROC permite que qualquer usuário possa visualizar processos em execução (isso inclui arquivos abertos) de outros usuários, incluindo processos do usuário ROOT!

Basta executar um $ ps aux que você verá os processos em execução de todos os usuários.

### Mas porquê isso é um problema?

Se um invasor conseguir acesso a shell do sistema (utilizando algum RCE por exemplo), ele pode ter acesso a informações de processos que usuários do sistema estejam utilizando. Em suma, isso pode ser considerado um vazamento de dados (data leak), que pode levar a complicações sérias.

#### Que tipos de complicações?

Este invasor com acesso a estas informações do sistema pode fazer uso de engenharia social para roubar mais dados ou extorquir o administrador do sistema.  
Como um usuário por padrão consegue ver processos em execução de outros usuários, vamos supor que o invasor resolva monitorar em tempo real os processos em execução no sistema:

![.](https://kali.seg.br/post/imgs/hidepid_tmux.webp)

Observem que em um momento, o invasor nessa situação hipotética capturou a informação de que o ROOT estava criando/alterando um arquivo sugestivo chamado “senhas_de_emails.txt“.

Em face disso, o invasor poderia fazer uso de engenharia social e enviar um email para o administrador da empresa com o seguinte conteúdo:

`CARO ADMINISTRADOR,
TOMEI CONTROLE DO SEU SISTEMA E TODOS OS SEUS ARQUIVOS, E TIVE ACESSO INCLUSIVE AO ARQUIVO
SENHAS_DE_EMAILS.TXT QUE VOCE MEXEU HOJE.
SIM, TIVE ACESSO A TODOOOS OS SEUS EMAILS, CONVERSAS PRIVADAS, SENHAS, TUDO. AH, E NEM ADIANTA
MUDAR AS SENHAS, PORQUE JÁ FIZ BACKUP DE TODOS OS EMAILS E ENCONTREI COISAS BEM
INTERESSANTES.
CLARO QUE POSSO ESQUECER TUDO ISSO SE VOCÊ ME DER UM INCENTIVO. QUE TAL ALGO EM TORNO DE 1
BITCOIN PARA O ENDEREÇO ABAIXO?
BTC: XXXXXXXXXXXXXXXXXXXXXXXX
SENÃO, ALGUMAS INFORMAÇÕES QUE DEVERIAM SER SIGILOSAS PODEM SER VAZADAS.
VOCÊ TEM 48 HORAS.`

Obviamente o invasor não teve acesso a nenhuma senha e nenhum e-mail, mas nesse exemplo hipotético, ele fez uso de uma informação que ele tinha (arquivo senhas_de_email.txt que foi acessado pelo root) e usou essa informação a seu favor para enganar o administrador, e contudo fazer ele acreditar que de fato o invasor teve acesso a essas informações. E como o administrador sabe que ele realmente alterou o arquivo senhas_de_emails.txt, ele automaticamente vai acreditar que todo o restante de informações ditas pelo invasor são também verdadeiras.

Quem trabalha com engenharia social, conhece todos esses mecanismos de PNL (programação neuro linguística).

## Como se proteger?

O Kernel possui uma funcionalidade que esconde informações de processos de outros usuários do sistema. Dando uma olhada no código fonte do Kernel (https://github.com/torvalds/linux), encontramos a seguinte informação:

![.](https://kali.seg.br/post/imgs/hidepid_source.webp)

### Nível Padrão – 0

Por padrão, o Kali Linux (e a maioria das distribuições Linux) mantém os processos visíveis para todos os usuários (HIDEPID = 0), o que conforme vimos é um risco de segurança, pois pode vazar informações privilegiadas para outros usuários.

### Nível Parcialmente Restrito – 1

É possível restringir um pouco essa visibilidade, para HIDEPID = 1.
Quando setado para 1 (HIDEPID_NO_ACCESS), a partição /PROC é montada de modo a permitir que outros usuários consigam saber o número dos processos de todos os usuários (PID) que estão em execução, mas não permite que se consiga ler a informação do usuário que está executando e nem do que está sendo rodado nesses processos.  
Veja abaixo como funciona essa restrição na prática:

![.](https://kali.seg.br/post/imgs/hidepid_hidepid_1.webp)

Observem que utilizando o utilitário ps/top/htop, meu usuário comum (devnull) já não conseguiu mais visualizar os processos abertos pelo root, apenas meus próprios processos. Porém investigando “na unha” a partição /proc, consigo visualizar a numeração de processos (PID) não iniciados por mim. Mas ao tentar visualizá-los, não consigo saber nem quem iniciou o processo e nem o que é o processo em questão, pois o /PROC agora me limita essa visualização.  
Esta opção já é boa, mas abre brechas para se caso houver uma vulnerabilidade que permita explorar o /PROC, se saiba a numeração dos processos em execução, e consequentemente, explorar seu conteúdo.

### Nível restrito – 2

Para restringir ainda mais, podemos alterar a visibilidade para HIDEPID = 2.
Quando setado para 2 (HIDEPID_INVISIBLE), a partição /PROC é montada para não permitir que os usuários sequer saibam números de processos que não sejam os do próprio usuário. Veja abaixo o mesmo cenário acima com o HIDEPID = 2:

![.](https://kali.seg.br/post/imgs/hidepid_hidepid_2.webp)

Observem agora, que apenas o número dos processos do usuário logado constam na partição /PROC, todos os demais permanecem invisíveis.

Desta forma, notem que a partição /PROC passa a ter um comportamento diferente, a depender do usuário que estiver logado.

Esta opção já é restrita o suficiente, e ideal para a maior parte dos casos.

### Nível altamente restrito – 4

Chegamos no HIDEPID = 4 (ou HIDEPID_NOT_PTRACEABLE).  
Esta opção restringe os usuários de realizar até mesmo uma depuração ou rastreamento de system calls (chamadas do sistema), que envolvam PID que não sejam os iniciados pelo próprio usuário. Esta opção, por ser altamente restrita, não é recomendável para quem precise depurar aplicações e monitorar o funcionamento dela, ou rastrear as system calls. Recomendo ativar apenas em casos mais estritos e específicos.  
Como exemplo, se investigarmos as system calls do comando “ps aux” no nivel restrito (hidepid = 2), podemos visualizar o número de processos que não deveriam ser visíveis (abaixo):

![.](https://kali.seg.br/post/imgs/hidepid_hidepid_2_debug.webp)

Já utilizando o nível altamente restrito, nem isso é possível fazer.

## Como mudar essa opção?

Existem duas formas:

### Com o sistema em execução

Bastando para isso remontar a partição /PROC (executando como root ou elevando privilégios com o sudo/doas), informando o nível de restrição que se queira:

```bash
mount -o remount,rw,hidepid=2 /proc
```

Desse modo, remontamos a partição /PROC, com permissões de leitura e escrita (obviamente), e passamos o nível de restrição de visibilidade de processo que queremos.

Esse modo não é permanente, ou seja, sempre que o sistema for reiniciado, será necessário executar o comando novamente.

### Modo permanente

Dessa forma, o sistema já será sempre inicializado com a restrição de visibilidade de processos definida. Portanto alteramos o arquivo /etc/fstab (fstab vem de file system table), responsável por gerenciar os sistemas de arquivos e dispositivos que devem ser montados durante a inicialização do sistema.

```bash
proc    /proc        proc        defaults,hidepid=2    0 0
```

A linha acima no fstab, monta o sistema de arquivos “proc” no ponto de montagem /proc, mantendo o padrão (defaults) desse sistema de arquivos, acrescentando a restrição de visibilidade de processos (hidepid=2), ignorando flags de backup e verificação de integridade, que não fazem sentido em um procfs.

## Porque essa opção não vem restrita por padrão ao instalar o sistema? Existe contras?

Isso ocorre porque o Kali Linux (e a maioria das distribuições Linux) são abertas, para que se tenha uma alta funcionalidade no uso de ferramentas e processos.

Isso significa que uma restrição dessa visibilidade pode interferir no funcionamento de algumas ferramentas, principalmente as de monitoração de processos do sistema.  
Claro que é possível contornar isso, modificando as permissões do grupo de acesso utilizado por essas ferramentas, e cada caso deve ser avaliado isoladamente.

Lembrem-se que quanto mais segurança for parametrizado em um sistema, mais restrito ele é, e em alguns casos menos funcional ele fica.
