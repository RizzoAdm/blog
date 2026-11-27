---
title: "Backup em casa, parte 8: só a lista do que fica de fora (e uma trava para o dia em que o disco aparecer vazio)"
slug: "backup-parte-8-excecoes-e-trava"
date: 2026-12-01T08:00:00-06:00
draft: false
translationKey: "home-backup-part-8"
tags:
  - backup
  - restic
  - rclone
  - systemd
  - homelab
summary: "Troquei a lista do restic pela lista de exceções e descobri que isso abria um risco novo: um disco vazio também passa nas conferências. A trava que resolve isso, e a limpeza automática das versões na nuvem."
comments: true
---

Na [parte 7](/blog/pt-br/artigos/backup-parte-7-nuvem/), criei a regra de que todo backup pega a pasta inteira e só existe a lista de exceções. Na nuvem já era assim, mas o restic ainda usava a lista antiga, a do que entra. Era a vez dele.

## A lista nova

A lista antiga tinha oito pastas do disco principal e a minha home. A nova tem duas linhas:

- o disco principal inteiro (`/mnt/hd-dados`);
- o `/home` inteiro (neste PC só existe o meu usuário).

O resto foi para a lista de exceções: a lixeira, a quarentena, as cópias do catálogo e os jogos (que se reinstalam). Dos jogos, dois saves voltam. O restic só aceita trazer uma subpasta de volta (`!`) quando a pasta mãe é excluída com `/*`: `Jogos/*` funciona, `Jogos` não.

Também conferi as pastas que começam com ponto, que guardam as configurações dos programas. Elas entram, menos o cache, o que se reinstala e os segredos. (Achei mais um segredo: o arquivo de login de um programa. Entrou na lista de exceções, junto com as chaves SSH.)

Antes de trocar, rodei um dry-run (uma simulação que não grava nada). Deu 485.799 arquivos, contra 485.796 no snapshot de minutos antes. Ou seja, o conteúdo é o mesmo. O ganho é para o futuro: uma pasta nova entra sozinha.

## O risco que a lista nova criou

![IMAGEM: disco vazio passa nas conferências; a trava segura](/blog/images/home-backup-part-8/8a-trava.png)

Com a lista antiga, se o disco não montasse, `/mnt/hd-dados/RIZZO` não existia e o backup parava com erro. Com a lista nova, o caminho é `/mnt/hd-dados`, que é só o ponto de montagem e existe sempre, mesmo vazio.

O executor já conferia se o disco estava montado, então esse caso estava coberto. O que não estava coberto: o disco montado, mas com muita coisa apagada (um erro meu, um sync errado, um vírus pelo Windows). O restic faria um snapshot pequeno sem reclamar. Depois, a política de retenção (7 diários, 4 semanais, 6 mensais) iria trocando os snapshots bons pelos ruins, um por um, sem nenhum aviso.

Então, com o Claude, criei uma **trava de encolhimento**:

1. Depois de cada backup, o executor lê quantos arquivos o snapshot novo tem.
2. Compara esse número com o último aceito, guardado num arquivo de estado.
3. Se o snapshot novo tiver menos de 90% dos arquivos, ele **não roda a limpeza de snapshots**: o novo fica guardado e nenhum antigo é apagado.
4. Aparece um aviso na área de trabalho que **fica na tela até eu fechar**. Ele traz os comandos para conferir as pastas e, só se eu tiver apagado muita coisa de propósito, o comando para aceitar o tamanho novo.

O segundo dia ruim é bloqueado de novo, porque a referência continua sendo a contagem boa, e não a do dia anterior. Testei com um restic falso em 13 casos (disco vazio dois dias seguidos, exatamente 90%, 89,9%, liberação à mão...). O primeiro backup real com a trava passou: 485.796 arquivos, contra 485.681 de referência.

(Na hora de mandar o script para o PC, chegou uma versão anterior, porque reusei a mesma pasta temporária. Conferir o MD5 antes de instalar pegou o erro.)

## A limpeza das versões na nuvem

Na nuvem, o que o sync apagaria ou trocaria vai para uma pasta com a data, por exemplo `_versoes/2026-11-28/`. Faltava o script que apaga as pastas com mais de 6 meses. Algumas escolhas:

- **A idade vem do nome da pasta**, não da data dos arquivos. Um arquivo movido para as versões mantém a data original, então um arquivo de 2019 apagado ontem pareceria velho.
- **Vai para a lixeira do Drive**, que guarda por 30 dias. Ocupa cota por um mês, mas um erro tem volta.
- **Ele se recusa a apagar** se alguma pasta tiver data no futuro (relógio errado) ou se fosse apagar todas as pastas.
- Roda **uma vez por mês**, no dia 1 às 4h30, com a mesma trava dos outros jobs.

No primeiro teste de verdade, ele falhou: a pasta de versões ainda não existe, porque nenhum sync trocou arquivo até agora. Corrigi para tratar "pasta não existe" como "nada a apagar".

## O que isso não faz

- A trava não pega perdas pequenas. Apagar 5% de uma vez passa sem aviso.
- Ela conta arquivos, não confere o conteúdo. Um arquivo corrompido com o mesmo nome conta igual.
- Depois de uma limpeza grande de propósito, o backup da noite fica bloqueado até eu liberar à mão.
- A limpeza das versões depende do relógio do PC. Ela recusa datas no futuro, mas um relógio atrasado só faria apagar menos.

## Próximo passo

As cópias estão no lugar: o disco principal, o restic em casa e a pasta do Windows na nuvem. As fotos já ficam no Google Photos, e os trabalhos antigos ficam no arquivo frio. O que falta é provar que tudo isso volta. Até agora, o único teste de restauração que fiz foi pequeno: 19 arquivos sorteados, comparados pelo MD5. O próximo passo é restaurar uma pasta inteira, num lugar separado, e conferir arquivo por arquivo contra o original.
