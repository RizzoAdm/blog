---
title: "Backup em casa, parte 1: juntando 5 HDs antigos sem perder nenhum arquivo"
slug: "backup-parte-1-juntando-hds-antigos"
date: 2026-11-20T08:00:00-07:00
draft: false
translationKey: "home-backup-part-1"
tags:
  - backup
  - linux
  - python
  - sqlite
  - homelab
summary: "Início da série sobre a minha nova organização de arquivos e rotina de backup: um catálogo de tudo e o resgate de dois HDs com defeito."
comments: true
---

Olá a todos!

Nesta série de posts (sim, não vai ser um post único, pelo tamanho que o projeto está tomando), vou contar um pouco da minha nova organização de arquivos e pastas, e da nova rotina de backups que estou montando para os meus arquivos no computador pessoal, na nuvem, nos HDs externos etc.

Como eu não tenho um servidor NAS dedicado, todos os meus arquivos ficam no disco interno do meu desktop e em alguns HDs externos. E também tenho muita coisa guardada na nuvem (Google Drive).

Meus principais objetivos com esse projeto são: ter uma rotina de backup confiável e segura, e implementar o "backup 3-2-1" (3 cópias, 2 mídias diferentes, 1 cópia fora de casa, o famoso *offsite*) nos principais documentos e arquivos.

Uma regra vale para a série inteira: **nada é apagado porque "parece" duplicado. Só sai quando um hash prova que existe outra cópia idêntica.** (O hash, no meu caso o MD5, funciona como uma impressão digital do conteúdo: se dois arquivos têm o mesmo MD5, eles são idênticos, não importa o nome.)

Ferramentas usadas: Ubuntu, Python (só a biblioteca padrão), SQLite, rsync, rclone e smartmontools.

## 5 de novembro: um catálogo de tudo

![IMAGEM: catálogo dos discos e do Google Drive](/blog/images/home-backup-part-1/1a-catalogo.png)

O primeiro passo foi saber exatamente o que eu tenho e onde.

- **Um catálogo por conteúdo.** Mais de 1,4 milhão de arquivos, de todos os discos e do Google Drive, cada um registrado num banco SQLite com caminho, tamanho, data de modificação e MD5. Zero erros de hash. Se dois arquivos são duplicados, quem decide é o hash, nunca o nome.
- **Importar só o que é único.** O catálogo gera a lista do que copiar (uma cópia de cada conteúdo, sem arquivos auxiliares), o rsync copia, e cada arquivo é conferido de novo pelo MD5 no destino. Resultado: 13.886 de 13.886 batendo.
- **Google Fotos.** A API do Google Fotos não deixa mais ferramentas de terceiros acessarem a biblioteca do usuário. Então o caminho foi usar exportações agendadas do Google Takeout, entregues no Drive, baixadas com verificação e catalogadas: 358 GiB, 179.195 arquivos.
- Antes de apagar uma exportação antiga, comparei com a nova por hash. **6,9 GiB existiam só na antiga** e foram resgatados primeiro.

E o que ele não faz: o sistema nunca apaga nada sozinho. Toda remoção passa por um relatório, pela minha revisão e por uma quarentena com data.

## 7 de novembro: resgatando dois HDs com defeito

![IMAGEM: resgate dos dois HDs com defeito](/blog/images/home-backup-part-1/1b-resgate-hds.png)

Dois dos HDs antigos estão morrendo. Eles guardam backups pessoais antigos e anos de trabalho com 3D: projetos de clientes, bibliotecas de assets, texturas e renders.

- **Seagate ST2000DM001** (2 TB, 3,5"): 14.012 erros de CRC na interface e histórico de superaquecimento.
- **Seagate ST4000LM024** (4 TB, 2,5", SMR): falhou no teste SMART estendido aos 20%, com 8 setores pendentes depois de só 178 horas ligado.

A regra aqui foi: **ler o mínimo possível, mas provar tudo.**

1. **Metadados primeiro.** Li só o índice do NTFS (nome, tamanho, data) e comparei com o catálogo, que já tinha o MD5 de tudo que estava nos discos saudáveis. 99% bateu sem precisar abrir o conteúdo dos arquivos.
2. **Copiar primeiro o que está em risco.** Tudo que não bateu, mais todos os arquivos de texto e configuração: 21.580 arquivos, 217 GiB, cada um conferido pelo hash no destino.
3. **Medir o palpite.** Amostras aleatórias com hash mostraram de 0,6% a 1,5% de falsos positivos: metadados de assets, arquivos do Substance e até cenas `.blend` regravadas com o mesmo tamanho e a mesma data. Metadado é pista, não prova.
4. **Hash completo, só leitura.** Os 593.830 arquivos que "bateram" foram lidos nos discos com defeito e comparados com as cópias de referência, em 14,5 horas. No total, 1.228 eram diferentes e foram copiados.

Resultado: **22.805 arquivos únicos (222,4 GiB) resgatados**, todo o resto provado idêntico pelo MD5 a uma cópia em outro disco, 0 erros de leitura, e os contadores SMART sem mudança depois da leitura completa.

Na mesma semana, um disco saudável: 1.389,9 GiB (217.215 arquivos) de um Seagate ST4000DM004 importados para um IronWolf de 8 TB (ST8000VN004). As pastas de 3D foram copiadas como projetos inteiros, para os arquivos `.blend` não perderem as texturas. Conferidos pelo MD5: 217.215 de 217.215.

## O que isso não faz

- **Não conserta o disco.** Setor ruim é defeito físico; remapear só esconde o problema.
- **Não apaga nada.** Os discos antigos ficam intactos até uma conferência final provar que cada arquivo existe em outro lugar.
- **Juntar tudo num disco não é backup.** O backup criptografado 3-2-1 com restic vem depois.

Na [parte 2](/blog/pt-br/artigos/backup-parte-2-apagar-so-com-prova/), a limpeza: como apagar e deduplicar sem depender de palpite.
