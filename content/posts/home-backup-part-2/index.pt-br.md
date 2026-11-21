---
title: "Backup em casa, parte 2: apagar só com prova (limpeza, deduplicação e reorganização)"
slug: "backup-parte-2-apagar-so-com-prova"
date: 2026-11-23T08:00:00-07:00
draft: false
translationKey: "home-backup-part-2"
tags:
  - backup
  - linux
  - python
  - sqlite
  - homelab
summary: "Depois de juntar os discos, a limpeza: recatalogar tudo, decidir qual cópia fica e reorganizar 2,75 TiB sem perder o rastro de nenhum arquivo."
comments: true
---

Na [parte 1](/blog/pt-br/artigos/backup-parte-1-juntando-hds-antigos/), juntei o conteúdo de cinco HDs antigos num disco de 8 TB. Agora vem a parte delicada: limpar. Apagar é fácil; apagar sabendo que não perdi nada é outra história.

A regra continua a mesma: nada é apagado porque parece redundante. É apagado porque um hash diz que é.

Ferramentas desta parte: Ubuntu, Python (só a biblioteca padrão), SQLite, rsync, rclone e 7-Zip.

## 7 de novembro: prove antes de apagar

![IMAGEM: recatalogação e limpeza com prova](/blog/images/home-backup-part-2/2a-prove-antes-de-apagar.png)

Depois de resgatar os dois discos com defeito e importar um terceiro, o disco principal tinha 2 TiB de conteúdo novo. Antes de remover qualquer coisa, o catálogo precisava descrever o disco como ele está agora, não como estava uma semana antes.

1. **Recatalogar tudo.** Uma passada completa de metadados no IronWolf de 8 TB: 933.033 arquivos, 4,88 TiB, 0 erros. A área de importação bateu exatamente com os seis manifestos de importação: 437.239 arquivos.
2. **Calcular o hash do que é novo.** 452.970 arquivos (2,18 TiB) em 4h06, com 0 erros de leitura, 0 arquivos alterados durante a leitura e nenhum sem hash.
3. **Explicar cada diferença.** 79.910 linhas do catálogo não existiam mais no disco. Todas vinham de duas mudanças registradas: um lote da quarentena que já tinha sido apagado e arquivos que o sync do Google Drive tinha separado. Nenhuma perda sem explicação.
4. **Apagar com evidência.** 356 GiB de arquivos do Google Takeout só foram removidos depois de confirmar que os 179.195 arquivos extraídos estavam no disco com o mesmo MD5.
5. **Lixo vai para a quarentena, mas conferido antes.** 22,8 GiB de fragmentos de upload do app do Drive, arquivos `desktop.ini` e atalhos da nuvem foram para a quarentena, com uma checagem de "isso é único?" antes de cada movimentação. Uma lixeira antiga com 2.229 arquivos tinha 19 itens únicos, que ficaram para eu decidir.

Toda remoção segue o mesmo caminho: **relatório → revisão → quarentena → apagar**, com registro e um jeito de desfazer.

## 9 de novembro: qual cópia fica?

![IMAGEM: deduplicação com prova](/blog/images/home-backup-part-2/2b-deduplicacao.png)

Ponto de partida: 4,69 TiB no disco, com 1.038,8 GiB de duplicados exatos mapeados pelo MD5.

Achar duplicado é fácil. O problema de verdade é decidir **qual cópia fica**.

1. **Pastas antes de arquivos.** Uma pasta só sai se cada arquivo dela tiver uma cópia idêntica fora dela, num lugar que vai continuar existindo. Pastas "despejo" (backups de celular, pastas de "refazer") saem primeiro; a árvore organizada sai por último. Uma regra de nível impede trocar uma pasta organizada por uma cópia que está no meio da bagunça. Resultado: 331,4 GiB numa única árvore antiga de backups.
2. **Arquivo por arquivo só onde a pasta não decide.** Dentro das pastas despejo, um arquivo só sai se o mesmo conteúdo ficar numa pasta mais organizada, com o hash recalculado nos dois lados antes de mover: 16,4 GiB.
3. **Respeitar como eu organizei.** Numa exportação do Google Fotos, as cópias dos álbuns ficam e as das pastas por ano saem: 30.610 arquivos, 75,2 GiB.
4. **Projeto 3D precisa de um gêmeo, não de cópias espalhadas.** Uma pasta de projeto só sai se uma outra pasta tiver todos os arquivos dela. Saíram 84 gêmeos exatos e 26 gêmeos de conteúdo (23,3 GiB); 6 projetos cujos arquivos só existem espalhados ficaram. Uma primeira versão sem prioridades teria apagado 10,1 GiB da minha biblioteca de assets organizada para manter a cópia bagunçada. Eu peguei isso na revisão e mudei a regra.
5. **Abrir os compactados antes de apagar.** 530 arquivos ZIP, 7z e RAR tiveram o hash calculado arquivo por arquivo lá dentro. 89 (8,7 GiB) saíram porque tudo o que tinham já estava na biblioteca de assets. 35 cujo conteúdo só existe dentro de projetos de clientes ficaram, para entrar na biblioteca depois. Os que não abriram também ficaram.

Cada passo: **dry-run → minha revisão → quarentena → conferência no disco → apagar**. No final, **456,6 GiB de duplicados removidos**, e uma nova varredura encontrou 736.051 arquivos, exatamente o número esperado, com 0 erros.

Algumas coisas que ficaram claras:

- **Só vale MD5 idêntico.** Nada de "parecido".
- **O script não decide a organização.** Duplicados entre duas pastas igualmente organizadas (20,7 GiB) esperam por mim, na fase de reorganização.
- **Nome não é prova.** A importação de um dos discos com defeito, feita por nome, tamanho e data para não reler o disco, trouxe 5.104 arquivos `desktop.ini` do Windows como "únicos". Foram removidos pelo nome exato, e as próximas importações já excluem esse tipo de arquivo.

## 11 de novembro: mudando 2,75 TiB de lugar

![IMAGEM: reorganização do disco por assunto](/blog/images/home-backup-part-2/2c-reorganizacao.png)

Com o disco limpo, faltava organizar: transformar o conteúdo de cinco discos importados numa estrutura única, organizada por assunto. Foram **537.515 arquivos, 2,75 TiB**. O desafio era mover tudo e manter o catálogo exato.

1. **Renomear, não copiar.** Cada movimentação é um plano aprovado por mim: origem → destino. O dry-run classifica cada arquivo como mover, duplicado ou conflito, e a execução é recusada se sobrar um único conflito. Como é tudo no mesmo disco, os arquivos são movidos com `os.rename`: instantâneo, com as datas preservadas. O caminho no catálogo é atualizado no mesmo passo, e uma linha é escrita num diário antes de cada operação. Uma varredura completa depois não precisou recalcular nenhum hash e bateu exatamente com o esperado: 724.732 arquivos, 0 erros.
2. **Deixar os discos votarem.** Juntar as árvores de cinco discos gerou 4.687 grupos com o mesmo nome, tamanho e data, mas MD5 diferente. Onde os discos saudáveis tinham uma cópia, eles decidiram: 1.437 grupos eram edições reais (todas as versões ficaram). Corrupção silenciosa só apareceu no disco com defeito; essas cópias saíram e as boas ficaram.
3. **Nome engana.** Numa exportação do Google Fotos, IMG_1234 se repete entre câmeras e anos. Parear os arquivos auxiliares das fotos (*sidecars*) só pelo nome apagou 659 deles, de 8 álbuns. Nenhuma foto foi perdida, e eu vi a falha na revisão. Agora o pareamento exige o mesmo horário de captura, ao segundo, ou o mesmo MD5. Os 659 vão ser reconstruídos a partir de uma nova exportação, pelo MD5, e não adivinhados pelo nome.
4. **Compactados só saem quando o conteúdo existe extraído.** Cada arquivo de dentro com hash calculado e encontrado pelo MD5 numa pasta de assets ou de estudo. Cópias dentro de projetos de clientes não contam. 25 compactados (7 GiB) saíram e 10 ficaram.

Uma lição: **não confiar em script que move arquivos sem atualizar o catálogo.** Um dos primeiros lotes fez isso. Uma ferramenta de reparo reconciliou tudo, e desde então toda movimentação é registrada pela própria ferramenta ou reconciliada antes do passo seguinte.

## O que isso não faz

- **Não decide onde as coisas ficam.** A estrutura de pastas e as regras de conflito são decisões minhas, escritas antes da execução.
- **Não apaga em um passo só.** Tudo passa por relatório, minha revisão e quarentena.
- **Não "conserta" o que não consegue provar.** Quando não dá para provar, o arquivo fica e a decisão é minha.

Na [parte 3](/blog/pt-br/artigos/backup-parte-3-primeiro-backup-restic/), finalmente o backup: a primeira cópia completa com restic.
