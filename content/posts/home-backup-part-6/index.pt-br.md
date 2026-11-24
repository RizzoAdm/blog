---
title: "Backup em casa, parte 6: o mesmo vídeo duas vezes, com bytes diferentes"
slug: "backup-parte-6-quase-duplicatas"
date: 2026-11-27T08:00:00-07:00
draft: false
translationKey: "home-backup-part-6"
tags:
  - backup
  - python
  - sqlite
  - linux
  - homelab
summary: "Dois vídeos iguais na tela que o MD5 dizia serem diferentes. Como achei 1.654 quase duplicatas sem apagar nenhuma foto de verdade, e por que o nome do arquivo nunca decide nada."
comments: true
---

Na [parte 5](/blog/pt-br/artigos/backup-parte-5-escopo-e-disco-frio/), corrigi o escopo do backup e criei o disco frio. Desta vez, a pergunta veio enquanto eu olhava uma pasta de vídeos: por que `GOPR0006.MP4` e `GOPR0006(1).MP4` continuavam lá, se eu já tinha passado a deduplicação por MD5 no disco inteiro?

## Primeiro: as cópias idênticas com sufixo

![IMAGEM: o nome não decide, o conteúdo decide](/blog/images/home-backup-part-6/6a-nome-nao-decide.png)

A deduplicação que fiz na parte 2 trabalhava por pasta inteira: uma pasta só saía se tudo nela tivesse cópia em outro lugar. Arquivos repetidos dentro da mesma pasta (o famoso `foto(1).jpg` ao lado de `foto.jpg`) nunca entraram na conta.

Com a ajuda do Claude, escrevi um script para isso, com uma regra simples:

1. O arquivo tem um sufixo de cópia no nome: `(1)`, ` (2)`, ` - Cópia`, ` - Copy`. Um número de 4 dígitos, como `(2018)`, não conta (quase sempre é ano).
2. Existe na **mesma pasta** um arquivo com o nome sem o sufixo.
3. Os dois têm o **mesmo MD5** (a "impressão digital" do conteúdo).

Se as três coisas valem, a cópia vai para a quarentena e o nome original fica.

O resultado me surpreendeu. Dos 12.945 nomes com sufixo, só **1.226 (2,51 GiB)** eram cópias idênticas. Outros **7.720** tinham um "original" com o mesmo nome, mas com **conteúdo diferente**:

- câmeras e celulares reiniciam a numeração (`GOPR`, `IMG_`, `DSC`) quando trocam o cartão;
- o WhatsApp repete nomes;
- celulares nomeiam a foto pela data e hora, e duas fotos no mesmo segundo viram `20181003_165933.jpg` e `20181003_165933(0).jpg`.

Se eu tivesse apagado pelo nome, teria perdido milhares de fotos e vídeos de verdade.

## E o GOPR0006?

Esse par estava na lista dos "diferentes". Mas eu tinha assistido os dois, e eram iguais.

Fui ver de perto:

- tamanhos quase iguais: 9.552.522 e 9.565.817 bytes (0,14% de diferença);
- mesma data de gravação, até o segundo;
- mesma resolução (1920×1080) e codecs;
- duração 17,25 s contra 17,18 s.

Com o `ffmpeg`, calculei um hash só das trilhas de vídeo e áudio (sem o cabeçalho), e mesmo assim deu diferente. Nenhum dos dois tinha as etiquetas que a GoPro grava. Conclusão: era o mesmo clipe, enviado duas vezes para o Google Photos e recodificado a cada envio. Nenhum hash de arquivo acha isso, porque os bytes são diferentes de verdade.

## Quase duplicatas: a regra

![IMAGEM: data, câmera, duração e conferência visual](/blog/images/home-backup-part-6/6b-quase-duplicatas.png)

Montei, de novo com o Claude, um segundo script, só de relatório. Ele lê o cabeçalho de cada foto e vídeo com o `exiftool` (data de gravação, câmera, resolução, duração) e guarda tudo num banco SQLite separado, sem mexer no catálogo. Dois arquivos só formam um grupo se **tudo** bater:

- mesmo tipo (vídeo com vídeo, JPEG com JPEG; RAW nunca se mistura com JPEG);
- mesma data de gravação, até o segundo;
- mesmo modelo de câmera (fotos de dois celulares no mesmo segundo ficam separadas);
- mesma proporção da imagem;
- no vídeo, duração com até 0,5 s de diferença.

Só isso ainda não basta. Na primeira rodada, o script juntou duas fotos seguidas do mesmo celular, que o EXIF marcava com o mesmo segundo. Por isso entrou uma **conferência visual**: o `ffmpeg` reduz cada imagem a 64×64 em tons de cinza e mede a diferença média dos pixels (de 0 a 255).

Calibramos (eu e a IA) os limites com casos reais:

- a mesma foto recomprimida dá menos de 1;
- quadros seguidos de um time-lapse dão perto de 2,5;
- duas fotos de uma rajada dão de 5 a 27.

Nos vídeos, a primeira versão da comparação errou, e o Claude corrigiu a lógica depois que eu mostrei os números. Como as durações diferem um pouco, "o quadro a 50% do vídeo" caía em momentos ligeiramente diferentes, e numa cena de ação isso já muda a imagem. Agora, em cada ponto, o script procura o quadro mais parecido numa janela de ±0,35 s. Depois disso, todos os pares de vídeo iguais ficaram abaixo de 1.

Limites finais: foto abaixo de 1, vídeo abaixo de 3. Em cada grupo, fica o arquivo de maior qualidade (taxa de bits no vídeo, pixels na foto).

## O que eu decidi apagar

No disco inteiro, o relatório achou cerca de 10 mil pares "fortes" (mesma resolução) e 15 mil "fracos" (uma versão reduzida). Mas a maioria estava em **pastas diferentes**: a versão do Google Photos de um lado, a pasta organizada do outro. E uma regra minha, de antes, diz que a mesma foto pode estar em dois álbuns organizados.

Então fui conservador:

1. só dentro da **mesma pasta**;
2. só nas pastas de fotos e vídeos pessoais (nada de jogos, projetos 3D ou cursos);
3. só os grupos fortes e fracos (os "possível rajada" ficam).

Antes de executar, pedi ao Claude uma amostra de 10 pares escolhidos ao acaso e abri os dois arquivos de cada par lado a lado, por uma pasta de atalhos com nomes neutros (`01-forte-fica.mp4`, `01-forte-sai.mp4`). Estavam todos certos.

## Resultado

- **1.226** cópias idênticas com sufixo (2,51 GiB).
- **1.654** quase duplicatas na mesma pasta (8,39 GiB).
- Tudo na quarentena, com um diário de cada movimento e o catálogo atualizado. Antes de mover cada arquivo, o script relê o MD5 do que sai e do que fica.

São 10,9 GiB num disco de 8 TB. O ganho de espaço é pequeno, mas eu queria parar de ver a mesma foto duas vezes, e agora sei por que o MD5 não pegava esses casos.

## O que isso não faz

- Não compara fotos sem data no EXIF (prints, imagens do WhatsApp). Elas ficaram de fora: quase 89 mil arquivos.
- Não junta a mesma foto guardada em duas pastas diferentes. Isso é de propósito.
- Não pega versões editadas (recortadas ou com filtro). A proporção ou a imagem mudam, e o par não se forma.
- Uma pegadinha: o próximo Takeout do Google Photos vai trazer de volta as versões comprimidas que eu tirei, e vou ter de compará-las com o diário antes de importar.

## Próximo passo

Agora sim, o HD interno de 1 TB com a segunda cópia do núcleo. Ele vai ficar maior que o planejado: os vídeos que eu mesmo gravei e as pastas de trabalho que só existem aqui também precisam entrar.
