---
title: "Backup em casa, parte 5: o buraco que a lista de pastas abriu e o disco frio que se confere sozinho"
slug: "backup-parte-5-escopo-e-disco-frio"
date: 2026-11-26T08:00:00-07:00
draft: false
translationKey: "home-backup-part-5"
tags:
  - backup
  - restic
  - linux
  - sqlite
  - homelab
summary: "Meu backup rodava toda noite sem erro, e mesmo assim uma pasta importante estava fora dele. A correção, e um disco frio de 906 GiB conferido arquivo por arquivo."
comments: true
---

Na [parte 4](/blog/pt-br/artigos/backup-parte-4-google-drive/), eu tirei 326 GiB do Google Drive, sempre com uma cópia conferida no disco do acervo. Nesta parte, olhei de novo para o backup e encontrei um problema que eu mesmo criei.

Meu backup rodava toda noite, sem erro. E mesmo assim uma pasta importante estava fora dele.

## Como isso aconteceu

![IMAGEM: o backup por regra, e não por lista](/blog/images/home-backup-part-5/5a-escopo-por-regra.png)

Quando montei o job do restic (o programa de backup que uso), escrevi uma lista com as pastas do meu acervo: mídia, trabalho e estudos. Dias depois, juntei as pastas que estavam divididas entre o espelho do Google Drive e o acervo local, e isso criou uma pasta nova de documentos (inclusive finanças). Eu não coloquei essa pasta na lista. Depois, quando tirei da nuvem o que já tinha cópia local, a pasta de finanças passou a existir só no HD principal e nos snapshots antigos (um snapshot é a "foto" do backup de cada noite).

Achei o problema por acaso, medindo o tamanho das pastas para outra coisa.

## O que eu mudei

1. **Regra em vez de lista.** O backup agora grava a raiz do acervo inteira, menos o que eu excluo de propósito (hoje, só a biblioteca de assets 3D). Pasta nova entra sozinha.
2. **Snapshots agrupados por máquina e tag.** O restic, por padrão, procura o snapshot anterior pela lista de pastas. Se a lista muda, ele não acha o anterior e lê tudo de novo (horas), e a regra de quantos snapshots guardar passa a tratar os antigos como outro grupo, que nunca é limpo. Com `--group-by host,tags`, o snapshot anterior foi achado e o backup novo levou 4 minutos.
3. **Conferi dentro do backup.** Contei os arquivos de cada pasta dentro do snapshot e comparei com o meu catálogo. Bateu, menos 1.472 arquivos de `node_modules` (dependências de um projeto web antigo), que ficam fora de propósito.

De quebra, vi funcionar pela primeira vez a limpeza de snapshots antigos (o `prune`) do jeito que eu tinha configurado na parte 3.

## O disco frio

![IMAGEM: o disco frio de 2 TB, relido e conferido](/blog/images/home-backup-part-5/5b-disco-frio.png)

A biblioteca de assets 3D (905,9 GiB, 146.734 arquivos) não entra no restic. São coisas baixadas, em boa parte recuperáveis. Mesmo assim, eu queria uma segunda cópia.

Usei um HD antigo de 2 TB. Ele já teve erros de comunicação e superaquecimento no passado, mas a superfície está limpa. Escolhi esse disco justamente por isso: o disco mais fraco fica com o dado de menor valor, e o disco saudável de 1 TB fica com o que é mais importante.

1. Formatei com um script que acha o disco pelo número de série e se recusa a mexer em disco montado ou nos discos protegidos (o principal, o de backup e os SSDs).
2. Copiei com `rsync` em NTFS (assim qualquer Windows lê): 2 horas e meia, 0 erros.
3. Li tudo de volta: calculei o MD5 (uma "impressão digital" do conteúdo) de cada arquivo **lendo o disco frio** e comparei com o original, caminho por caminho. Resultado: 0 faltando, 0 diferentes, 0 sobrando.
4. Deixei o disco **se descrever sozinho**. Na raiz dele há uma pasta com um arquivo de instruções (os comandos de conferência logo no topo), a identidade do disco, um histórico do SMART (a autoavaliação de saúde do HD) que só cresce, e a lista de MD5 de todos os arquivos. Daqui a 3 meses eu ligo só esse disco e confiro tudo, sem depender de nada.

Uma lição de bônus: colocar o disco na dock dupla reiniciou a dock e desmontou o meu disco de backup, que estava na outra vaga. Não havia backup rodando, mas agora é regra: nunca mexer numa vaga com a outra em uso.

## O que isso faz e o que não faz

Faz: qualquer pasta nova do acervo entra no backup sem eu lembrar, e o disco frio pode ser conferido sozinho, em qualquer Linux.

Não faz:

- O disco frio não acompanha mudanças. É uma cópia com data, atualizada à mão.
- Ele não fica fora de casa. A cópia off-site é o próximo disco.
- Só os discos de cópia simples têm a lista de MD5. Os discos do restic são conferidos pelo próprio restic, e assim os nomes dos arquivos continuam criptografados.

## Próximo passo

O HD interno de 1 TB vai receber a segunda cópia do núcleo (documentos, finanças e fotos) num repositório restic próprio.

Antes disso, apareceu uma limpeza que não estava no plano: está na [parte 6](/blog/pt-br/artigos/backup-parte-6-quase-duplicatas/).
