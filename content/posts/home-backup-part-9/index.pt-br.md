---
title: "Backup em casa, parte 9: restaurei uma pasta inteira e conferi arquivo por arquivo"
slug: "backup-parte-9-teste-de-restauracao"
date: 2026-12-02T08:00:00-06:00
draft: false
translationKey: "home-backup-part-9"
tags:
  - backup
  - restic
  - homelab
summary: "Um backup só vale se volta. Restaurei 17 GiB de documentos num lugar separado e comparei o MD5 de cada arquivo com o original: 7.883 de 7.883 iguais, e um aviso que precisou de explicação."
comments: true
---

Na [parte 8](/blog/pt-br/artigos/backup-parte-8-excecoes-e-trava/), terminei dizendo que faltava provar que tudo volta. Até agora, o único teste de restauração que eu tinha feito era pequeno: 19 arquivos sorteados. Desta vez, restaurei uma pasta inteira.

## A pasta e o lugar

Escolhi a pasta de documentos do Acervo: 17 GiB e 9.355 arquivos, quase todos pequenos e muitos com acento ou caracteres estranhos no nome. É onde uma restauração costuma dar problema.

O destino foi uma pasta temporária nova no SSD, dentro da minha home. Não usei o `/tmp` porque no Ubuntu ele às vezes fica na memória RAM, e aí o teste não gravaria em disco de verdade. No fim, o script apaga a pasta sozinho, mas só se não houver nenhum problema.

## O que o teste confere

Com o Claude, aumentei o script do teste antigo para responder três perguntas:

1. **Voltou tudo?** O script lista o que o snapshot (a "foto" do backup) tem naquela pasta e confere se cada arquivo foi restaurado.
2. **O conteúdo é igual?** Compara o MD5 (uma impressão digital do arquivo) de cada restaurado com o original.
3. **E o que mudou depois do backup?** O snapshot é de um horário. Se eu editei um arquivo depois, ele vai ser diferente, e isso não é erro do backup. O script separa esses casos pela data de modificação.

O resultado cai em classes. Problema: conteúdo diferente sem o original ter mudado, ou arquivo que não voltou. Informativo: mudou depois, saiu da pasta, é novo. Aviso: um arquivo antigo que está na pasta mas não está no backup.

Antes de rodar de verdade, testei com um restic falso e 10.008 arquivos, inclusive um com quebra de linha no nome. Cada classe apareceu onde devia.

## Resultado

![IMAGEM: 7.883 de 7.883 arquivos iguais](/blog/images/home-backup-part-9/9a-resultado.png)

- Restauração: 7.883 arquivos em 2 min 28 s (cerca de 110 MiB/s, lendo do disco de backup na dock USB).
- Comparação: 2 min 16 s.
- **7.883 iguais (16,28 GiB). Nenhuma diferença, nenhuma falta, nenhum erro.**

E o aviso: **1.472 arquivos** estavam na pasta e não estavam no backup. Parecia um buraco. Mas eram todos de uma única pasta de projetos antigos, todos dentro de `node_modules` (as bibliotecas que o npm baixa para um projeto JavaScript). Esse nome está na minha lista de exceções de propósito. Pensei em abrir uma exceção para o arquivo morto e decidi manter a regra: o código-fonte dos projetos está no backup, só as bibliotecas ficam de fora.

(Na mesma madrugada rodaram as primeiras execuções agendadas: o restic às 2h e o sync da nuvem às 3h30. As duas terminaram sem erro.)

## O que isso não faz

- Testa uma pasta e um snapshot, não o backup inteiro.
- Não é um teste de desastre: não restaurei num PC novo, sem o disco principal.
- Não testa a cópia na nuvem.
- A separação entre "mudou depois" e "problema" depende da data de modificação. Um arquivo alterado sem mudar a data passaria como problema (o que é melhor do que o contrário).

## Próximo passo

O teste agora pode ser repetido em qualquer pasta. Falta decidir com que frequência rodar e fazer o teste maior, que é restaurar tudo num disco vazio. Antes disso, quero olhar pasta por pasta o que está duplicado entre o Acervo e a nuvem.
