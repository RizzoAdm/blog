---
title: "Backup em casa, parte 3: o primeiro backup com restic (de 19 para 93 MB/s, e a armadilha do prune)"
slug: "backup-parte-3-primeiro-backup-restic"
date: 2026-11-26T08:00:00-07:00
draft: false
translationKey: "home-backup-part-3"
tags:
  - backup
  - restic
  - linux
  - systemd
  - homelab
summary: "A segunda cópia do 3-2-1: 2,75 TiB num disco SMR, cinco vezes mais rápido depois de uma mudança, e um bug na retenção descoberto uma semana antes de acontecer."
comments: true
---

Com tudo organizado num disco só (veja a [parte 1](/blog/pt-br/artigos/backup-parte-1-juntando-hds-antigos/) e a [parte 2](/blog/pt-br/artigos/backup-parte-2-apagar-so-com-prova/)), chegou a hora do backup de verdade. Este post é sobre a **segunda cópia** do 3-2-1.

Ferramentas desta parte: Ubuntu, restic 0.18, systemd (serviço, timers e inibidor de suspensão), ext4 e Bash.

## 14 de novembro: o primeiro backup completo

![IMAGEM: primeiro backup com restic no disco SMR](/blog/images/home-backup-part-3/3-restic-smr.png)

- **Origem:** 458.458 arquivos no disco principal de 8 TB, mais a minha pasta home.
- **Destino:** um Seagate ST4000DM004 de 4 TB reaproveitado, numa dock USB 3, formatado em ext4, com um repositório restic (formato 2, compressão ligada).

Um detalhe importante: esse disco é **SMR** (gerenciado pelo próprio disco). SMR é uma tecnologia que deixa o disco mais barato e com mais capacidade, mas que fica muito lenta quando as gravações não são sequenciais.

### 1. Achar o gargalo antes de mexer em qualquer coisa

Depois de uma primeira meia hora rápida, o backup estabilizou em 23 MB/s, com previsão de 30 horas. Uma amostra de 20 segundos do `/sys/block/*/stat` respondeu a pergunta:

- o disco de destino gravava 19 MB/s, com 97% de ocupação;
- o disco de origem lia com só 11% de ocupação;
- o barramento estava em 5 Gbps e a CPU, 96% ociosa.

O disco SMR estava engasgando com gravações fora de ordem: dois *packs* sendo gravados em paralelo, mais o journal do ext4.

### 2. Deixar as gravações sequenciais

Parei o backup e reiniciei com `-o local.connections=1 --pack-size 64`. Reiniciar custou pouco: o restic é endereçado por conteúdo, então os 468 GiB que já estavam gravados foram relidos em 40 minutos e nada foi gravado duas vezes.

Depois da mudança: **93,5 MB/s gravando, com 87% de ocupação**. O backup inteiro levou 15h45, com trechos rápidos (105–126 MB/s) e trechos longos mais lentos (31–41 MB/s).

### 3. A armadilha do prune

Ao limpar os 600 *packs* órfãos que sobraram da execução interrompida (10 GiB), o `restic prune` manual se recusou a rodar: **o prune exige pelo menos duas conexões**.

O job agendado roda `forget --prune` com as mesmas opções do backup. Só que o `forget` só faz o prune quando realmente esquece algum snapshot. Ou seja: o job tinha passado hoje e ia falhar mais ou menos uma semana depois, no primeiro dia em que a política de retenção entrasse em ação.

A correção: a conexão única agora vale só para o comando de backup, e o job recusa logo no início qualquer configuração que quebraria o prune. Melhor descobrir antes de horas de backup do que depois.

### 4. Prova, não suposição

- 2,749 TiB de origem armazenados em 2,162 TiB (graças à compressão e à deduplicação do restic).
- Arquivos aleatórios de todas as áreas, mais um banco de dados de 1,4 GB, foram extraídos com `restic dump` e comparados pelo MD5: **19 de 19 idênticos**.
- O `restic check` leu 1% dos *packs* de dados (538) com 0 erros.

### 5. Automação que combina com o meu uso

A dock só fica ligada na hora do backup. Então:

- **montar o disco já dispara o backup** (um serviço do systemd ligado à unidade de montagem);
- um timer às 02:00 recupera o backup perdido quando o computador liga;
- uma checagem diária separada, que não precisa do disco de backup, avisa depois de 7 dias sem um backup com sucesso, mostrando quantos arquivos ainda estão sem backup;
- o root segura o inibidor de suspensão (para o computador não dormir no meio), e o restic roda com o meu usuário, nunca como root.

## O que isso não faz

- **Não transforma SMR num bom destino de backup.** Execuções longas ainda ficam lentas, e eu não medi a causa dos trechos lentos (arquivos pequenos ou o cache do disco enchendo). Aqui é aceitável porque os backups diários são incrementais.
- **Não vale para qualquer caso.** Uma conexão e *packs* de 64 MiB deixaram *este* disco, *nesta* dock, mais rápido. Os números são só deste setup.
- **Não prova o repositório inteiro.** Ler 1% dos dados é uma amostra; a leitura completa precisa entrar na rotina.
- **Ainda não é 3-2-1.** Esta é a cópia dois. A segunda cópia local e a cópia na nuvem são as próximas fases.

Na [parte 4](/blog/pt-br/artigos/backup-parte-4-google-drive/), o Google Drive: juntar pastas divididas, um bug de nomes duplicados e 326 GiB fora da nuvem.
