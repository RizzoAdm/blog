---
title: "Backup em casa, parte 7: a cópia que sai de casa (e uma regra que eu devia ter criado no começo)"
slug: "backup-parte-7-nuvem"
date: 2026-11-30T08:00:00-06:00
draft: false
translationKey: "home-backup-part-7"
tags:
  - backup
  - rclone
  - google-drive
  - systemd
  - homelab
summary: "Desisti do terceiro disco em casa e mandei a pasta do meu perfil do Windows para o Google Drive: cópia simples, conferida arquivo por arquivo, todo dia às 3h30, com uma permissão que não deixa o script tocar no resto do Drive."
comments: true
---

Na [parte 6](/blog/pt-br/artigos/backup-parte-6-quase-duplicatas/), achei as quase duplicatas e mandei tudo para a quarentena. O plano agora era o HD interno de 1 TB, com a segunda cópia do "núcleo". Não foi o que aconteceu.

## Primeiro, a quarentena

Antes de apagar, conferi no disco, contra o diário de cada lote, que:

- todo arquivo da quarentena estava lá;
- todo original que fica ainda existia;
- nada tinha voltado para o lugar antigo.

Também pedi mais uma amostra de 20 pares das quase duplicatas: os 10 com maior diferença visual (os mais arriscados) e 10 sorteados. Eram todos a mesma foto. Apaguei 2.880 arquivos (cerca de 11 GiB) e o catálogo foi junto.

(O script da amostra parou na primeira vez sem dizer nada. Com poucas linhas de teste ele funcionava, mas com 1.200 linhas o `sort | head` morre por SIGPIPE, que é o aviso de "ninguém está mais lendo". A lição: testar com volume de verdade.)

## O 1 TB que não cabia

![IMAGEM: o que cabe no 1 TB](/blog/images/home-backup-part-7/7a-nucleo-nao-cabe.png)

Medi pelo catálogo o tamanho **único** de cada combinação de pastas (cada MD5 contado uma vez, como o restic faz). O núcleo básico (documentos, a pasta do Windows, os arquivos de outro computador e a home do Linux) dava 464 GiB, metade do disco. Qualquer coisa a mais estourava:

- com os trabalhos 3D: 1.208 GiB;
- com as fotos e vídeos sem o Google Photos: 971 GiB.

Então mudei de ideia. Duas cópias em casa (o disco principal e o disco do restic) bastam. O que falta no 3-2-1 é **a cópia fora de casa**, e para isso eu já pago 2 TB de Google Drive, com 1,1 TiB livre.

## A pasta do Windows na nuvem

Comecei pela pasta do meu perfil do Windows (Documentos, Imagens, Vídeos): 52,6 GiB. Algumas escolhas:

1. **Cópia simples, legível no Drive.** Abro qualquer arquivo pelo celular, sem restaurar nada. O preço é que não há criptografia.
2. **`rclone sync` com pasta de versões.** A nuvem fica igual ao disco, mas o que seria apagado ou sobrescrito vai para uma pasta com a data (`_versoes/2026-11-27/`) em vez de sumir. Versões com mais de 6 meses saem.
3. **Uma permissão nova e estreita.** O rclone que lê o Drive para o espelho local só tem permissão de leitura. Para enviar, criei um segundo acesso com a permissão `drive.file`: ele só enxerga os arquivos que ele mesmo criou. Para conferir, listei o Drive por esse acesso antes de criar a pasta de destino: veio vazio. Um comando errado não tem como apagar as pastas compartilhadas.
4. **Cuidado com o espelho.** O espelho local baixa o Drive inteiro, então ia trazer a cópia de volta para o disco. Acrescentei uma exclusão no comando dele.

O primeiro envio levou umas 5 horas (35.494 arquivos, 0 erros). Depois, o `rclone check` recalculou o MD5 de cada arquivo local e comparou com o MD5 que o Google guarda: **35.494 iguais, 0 diferenças**.

## Uma regra que eu devia ter criado no começo

Na [parte 5](/blog/pt-br/artigos/backup-parte-5-escopo-e-disco-frio/), uma pasta importante ficou fora do backup porque a lista do restic dizia o que **entra**, e a pasta foi criada depois da lista. Agora a regra vale para tudo: **o backup pega a pasta inteira, e a única lista é a de exceções** (o que fica de fora). Na nuvem já é assim: o arquivo de exceções tem `Downloads`, uma cópia antiga de documentos e os arquivos auxiliares do Windows. Qualquer pasta nova entra sozinha.

## Todo dia, sem eu lembrar

![IMAGEM: restic às 2h, nuvem às 3h30, a mesma trava](/blog/images/home-backup-part-7/7b-agenda.png)

Com o Claude, montei um executor no mesmo modelo do restic:

- um timer do systemd às 3h30, que roda quando eu ligar o PC se ele estiver desligado nesse horário;
- **a mesma trava do restic**: se os dois coincidem, um espera o outro, e o disco nunca é lido em dobro;
- ele **se recusa a rodar se a pasta de origem estiver vazia** (um sync de uma pasta vazia mandaria a cópia inteira da nuvem para as versões);
- um aviso na área de trabalho se passar 7 dias sem um envio bem-sucedido.

Testei tudo com um rclone falso antes de rodar de verdade. Na primeira execução pelo systemd, sem nada novo para mandar, levou 49 segundos.

## O que isso não faz

- Não criptografa. Quem tem acesso à conta Google lê os documentos.
- Não guarda versões para sempre: depois de 6 meses, um erro antigo só tem conserto pelo restic, em casa. (O script que apaga as versões antigas ainda vou escrever. Ele vai pela data no nome da pasta, porque o arquivo movido mantém a data original.)
- Não cobre o resto do disco. As fotos e os trabalhos têm duas cópias, as duas em casa.
- O restic ainda usa a lista antiga, do que entra. Ele viola a regra nova.

## Próximo passo

Trocar a lista do restic pela lista de exceções: o disco inteiro, menos o que eu escolher deixar de fora.
