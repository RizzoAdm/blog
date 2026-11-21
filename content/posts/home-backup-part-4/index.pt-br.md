---
title: "Backup em casa, parte 4: arrumando o Google Drive e tirando 326 GiB da nuvem"
slug: "backup-parte-4-google-drive"
date: 2026-11-30T08:00:00-07:00
draft: false
translationKey: "home-backup-part-4"
tags:
  - backup
  - google-drive
  - rclone
  - python
  - sqlite
  - homelab
summary: "Pastas divididas entre o disco e a nuvem, um espelho do Drive que 'apagava' documentos sozinho, e 326 GiB tirados da nuvem sem perder nenhum byte."
comments: true
---

Nas partes anteriores, organizei os discos ([parte 1](/blog/pt-br/artigos/backup-parte-1-juntando-hds-antigos/) e [parte 2](/blog/pt-br/artigos/backup-parte-2-apagar-so-com-prova/)) e fiz o primeiro backup com restic ([parte 3](/blog/pt-br/artigos/backup-parte-3-primeiro-backup-restic/)). Faltava o Google Drive.

Primeiro, como funciona o meu espelho do Drive:

- O meu acervo principal fica no disco de 8 TB. O Google Drive é uma área de trabalho, não o lugar "oficial" dos arquivos.
- O Drive é copiado num sentido só, da nuvem para o disco, com `rclone sync`.
- Qualquer arquivo que o sync apagaria no disco não é apagado: a opção `--backup-dir` move ele para uma **pasta de separados**, com data, e depois eu reviso.
- O catálogo SQLite tem caminho, tamanho, data e MD5 de cada arquivo do disco. Então "eu ainda tenho isso em algum lugar?" é uma consulta, não uma busca.

Ferramentas desta parte: Ubuntu, Python (só a biblioteca padrão), SQLite, rclone e Bash.

## 15 de novembro: cada pasta inteira, num lugar só

![IMAGEM: juntando pastas divididas entre o acervo e a nuvem](/blog/images/home-backup-part-4/4a-uma-pasta-uma-casa.png)

Quando importei os discos antigos, todo arquivo que já tinha uma cópia idêntica no espelho do Drive foi pulado, como "já tenho". O resultado: o acervo ficou com **pastas-esqueleto**. Pastas de projeto que só tinham os arquivos que não existiam em nenhum outro lugar, enquanto o resto morava só no espelho. Nada foi perdido, mas nenhum lugar tinha o projeto completo.

A regra agora é simples: **cada pasta mora inteira num lugar só, o acervo.** A nuvem recebe o que eu escolher subir.

1. **Medir antes de planejar.** O espelho tinha 37.766 arquivos (748 GiB). Sincronizei o que estava pendente e recatalogei o disco antes de planejar qualquer coisa.
2. **Descobrir onde cada pasta da nuvem mora no acervo.** Para cada arquivo do espelho com cópia no acervo, o caminho mais parecido gera uma regra, do tipo "pasta X da nuvem → Acervo/Trabalho/3D/X". A regra só vale se tiver pelo menos 3 arquivos e explicar pelo menos metade dos arquivos daquela pasta. Comparar pelo caminho, e não só pelo conteúdo, é o que revela o esqueleto: a pasta existe dos dois lados, mas quase não tem arquivos em comum.
3. **Conteúdo não é endereço.** Na primeira tentativa, pastas de viagem foram pareadas com a exportação do Google Fotos, e material de curso com a biblioteca de assets 3D, porque essas áreas têm os mesmos arquivos. Agora elas contam como cópia, nunca como "casa" de uma pasta.
4. **Eu escolho a casa, e a pasta vai inteira.** Doze pastas da nuvem ganharam uma casa no acervo, incluindo áreas novas para documentos. Cada projeto é copiado com a própria estrutura, em vez de ser espalhado.
5. **Duas cópias da mesma pasta da empresa viram uma.** A minha pasta e uma pasta compartilhada tinham basicamente o mesmo conteúdo reorganizado: 737 caminhos idênticos, cerca de 2.800 arquivos em caminhos diferentes e 1.408 só na compartilhada. A compartilhada foi juntada na minha: 2.315 arquivos (21 GiB) pulados porque já estavam lá, os exclusivos entraram no mesmo caminho relativo, e 4 conflitos de nome ficaram para revisão.
6. **Arquivos nativos do Google.** O sync já exporta Docs, Sheets e Slides para docx, xlsx e pptx. Os 88 arquivos nativos foram conferidos com as cópias exportadas; os 4 que não podem ser exportados (formulários e um mapa) ficaram marcados para exportação manual.
7. **Copiar com prova.** O plano: 28.341 arquivos (690 GiB). A ferramenta de cópia confere cada linha com o catálogo e o disco, escreve no diário antes de copiar, copia para um arquivo temporário calculando o hash, relê do disco e calcula o hash de novo, mantém a data original e só dá o nome final se o nome estiver livre. Nunca sobrescreve e pode ser rodada de novo sem problema.

## 16 de novembro: um nome, dois arquivos

![IMAGEM: o espelho do Google Drive que "apagava" documentos](/blog/images/home-backup-part-4/4b-um-nome-dois-arquivos.png)

A junção terminou: 35.378 arquivos, 727 GiB, cada byte relido e conferido. O último passo antes de limpar a nuvem era rotina: rodar o sync, olhar o dry-run e seguir.

**O dry-run disse que 28 documentos tinham sido apagados na nuvem. Ninguém tinha apagado nada.**

Meia hora depois, outro dry-run queria copiar 16 arquivos de volta para a mesma pasta, sem nada ter mudado na nuvem. O log tinha a resposta, 274 vezes:

```
Duplicate object found in source - ignoring
```

O Google Drive não é um sistema de arquivos: **dois arquivos com o mesmo nome podem ficar na mesma pasta.** O rclone precisa transformar a nuvem numa árvore de caminhos, então, para cada nome duplicado, fica com um e ignora o outro. E qual ele escolhe depende da ordem em que a API devolve os arquivos, que não é sempre a mesma. Se os dois têm conteúdo diferente, a cada sync um substitui o outro. Nada se perde, mas a pasta de separados enche de fantasmas.

**Medi primeiro.** O `rclone dedupe --dedupe-mode list` só lê e lista os grupos de nomes repetidos: 272 grupos, 186 idênticos (upload repetido, principalmente em duas pastas de fotos) e 86 com conteúdo diferente.

**Depois, o erro.** O modo `--dedupe-mode skip` apaga só os gêmeos idênticos e manda para a lixeira do Drive (30 dias para desfazer). Como o meu espelho é só leitura, criei um segundo acesso, com permissão de escrita, só para essa operação. Dry-run: 178. Execução: 178, sem erros.

Aí, conferindo um dos pares: o arquivo que ficou na pasta era um **atalho**. O arquivo de verdade estava na lixeira.

O rclone segue os atalhos do Drive, e um atalho tem o mesmo tamanho e o mesmo MD5 do arquivo para onde aponta. Para o dedupe, "um arquivo e um atalho para ele" é igual a "dois arquivos idênticos", e ele não tinha motivo para preferir o verdadeiro. Restaurei os 178 da lixeira pela web. Nada foi perdido (a lixeira fez o trabalho dela), mas foi preciso refazer.

**A correção.** Com `--drive-skip-shortcuts`, o rclone ignora os atalhos. A mesma lista agora mostrou 253 grupos e 174 pares idênticos de arquivos reais. O dedupe rodou de novo: 174. Depois, um script conferiu cada arquivo apagado: o gêmeo idêntico continua na nuvem? Continua no espelho? Existe uma cópia com o mesmo MD5 no acervo? **174 / 174 / 174.** O acesso com escrita foi apagado na mesma hora.

**A segunda causa.** Uma pasta de 28 arquivos continuava trocando de lugar. Dessa vez, o mesmo ID de pasta aparecia em dois caminhos: o real e um **atalho de pasta** dentro do meu próprio Drive. O rclone colocava os arquivos ora num caminho, ora no outro. Apaguei o atalho pela web e acabou.

**Fechando.** Com o espelho estável, limpei a pasta de separados: para cada arquivo, um script procura no acervo uma cópia com o mesmo MD5, **relê os dois arquivos do disco**, e só então apaga. 198 saíram. Os 78 que sobraram eram únicos (versões antigas de documentos ainda em uso na nuvem e alguns arquivos de trabalho apagados de lá anos atrás). Decidi guardar todos no acervo. Quando já existia um arquivo diferente com o mesmo nome (a versão atual), a versão antiga foi copiada ao lado, com o sufixo `(Drive <data>)`. Antes de rodar no disco real, ensaiei a sequência inteira num disco e num catálogo de teste, e o ensaio falhou duas vezes, justamente nas passagens de um script para o outro. Melhor no teste do que no acervo.

**Quatro regras que saíram disso:**

1. **Nunca rodar `rclone dedupe` no Google Drive sem `--drive-skip-shortcuts`.** Um atalho tem o hash do arquivo para onde aponta e sempre vai parecer idêntico a ele.
2. **Acesso de escrita na nuvem é temporário.** O espelho continua só leitura; o acesso com escrita existe para uma operação e é apagado assim que ela é conferida.
3. **Uma exclusão é conferida depois de acontecer, não só antes.** O número do dry-run bater com o da execução é necessário, mas não basta. Para cada arquivo removido, provar onde estão as cópias.
4. **Ensaiar a sequência inteira, não só um passo.** Cada script estava certo sozinho; as falhas estavam na passagem de um para o outro.

Os 174 duplicados removidos liberaram só 0,38 GiB. O objetivo nunca foi espaço: um espelho que fica trocando arquivos não consegue dizer o que realmente foi apagado na nuvem, e essa é a única pergunta que um backup precisa responder.

## 17 de novembro: 326 GiB fora do Google Drive

![IMAGEM: 326 GiB tirados do Google Drive](/blog/images/home-backup-part-4/4c-326-gib-fora-da-nuvem.png)

Com o espelho estável, chegou o passo para o qual toda essa junção foi feita: **diminuir a nuvem**.

A regra foi escrita antes de começar: **nada sai do disco de 8 TB, a não ser lixo.** "Arquivo frio" quer dizer *fora da nuvem*, não *apagado*. Uma pasta só sai do Google Drive quando cada arquivo dela está provadamente no acervo, byte por byte (e o acervo está no backup).

### Os gêmeos que não dá para escolher pela web

Sobraram seis grupos que nenhum script podia resolver: mesmo nome, mesma pasta, conteúdo diferente. Eram planilhas de 2010 e um plano de projeto de 2018. Em quatro deles, os gêmeos tinham **o mesmo nome, o mesmo tamanho e a mesma data**. A única diferença era o MD5, e a interface web do Drive não mostra o MD5. Não tinha como "escolher o certo" olhando.

Então mudei a pergunta: em vez de *qual deve ficar?*, *faz diferença qual fica?* Um script só de leitura conferiu as 13 versões: **as 13 tinham cópia no acervo.** Com todas seguras, a escolha na web deixou de ser um risco.

E isso fez diferença. Em dois pares as datas eram diferentes, e o plano era manter o mais novo. O dry-run discordou: as versões de 2010 tinham ficado e as de 2020 tinham ido para a lixeira. Coluna de data numa lista da web é um instrumento fraco. Nada se perdeu: a regra da pasta de separados encontrou as cópias no acervo, conferiu os dois lados e resolveu.

**A lição:** quando alguém precisa fazer uma escolha que não consegue conferir, primeiro faça a escolha não importar.

### 326 GiB, pasta por pasta

Cinco áreas saíram da nuvem: as pastas antigas de cursos de 3D (254 GiB), a parte da pasta da empresa de 3D que era da minha conta pessoal (46 GiB), a pasta de finanças, a de downloads e um projeto de vídeo de uma viagem de 2018 (25 GiB). Cada uma passou pelo mesmo ciclo:

1. **Provar as cópias.** Um relatório só de leitura mostrando quantos arquivos têm cópia com o mesmo MD5 fora do espelho, e onde. Todas as áreas deram 100%.
2. **Conferir atalhos** dentro das pastas que vão sair. Nenhum.
3. **Mandar para a lixeira pela web.** A lixeira do Drive guarda tudo por 30 dias.
4. **Dry-run.** Só pode aparecer um tipo de linha, "move into backup dir", e o número tem que bater com o relatório: 1.327 nos cursos de 3D, 4.312 na pasta da empresa, 4.251 nas outras três. Bateu.
5. **Sync.** Com o `--backup-dir` no mesmo disco, o rclone não apaga nada, só renomeia: 254 GiB movidos em 19,5 segundos.
6. **Registrar, conferir e apagar.** E foi aqui que apareceu um buraco no método.

### O buraco: o catálogo não sabia para onde os arquivos foram

O script que apaga os arquivos da pasta de separados procura cada um no catálogo pelo caminho. Só que o sync tinha acabado de mover os arquivos, e o catálogo ainda tinha eles no caminho antigo. Rodando como estava, o script ia dizer que nenhum dos 1.327 arquivos estava no catálogo e não ia apagar nada. Seguro, mas inútil. A única saída seria recatalogar 730 mil arquivos, para cada pasta.

A solução foi um modo novo no script de reparo do catálogo. Como o `--backup-dir` mantém o caminho relativo, cada arquivo separado tem uma linha correspondente no catálogo, com o mesmo tamanho e a mesma data. O script só move essa linha para o caminho novo e reaproveita o MD5 que já existia. Isso é seguro porque o MD5 antigo nunca é usado sozinho: o script de apagar continua relendo os dois arquivos do disco antes de remover qualquer coisa. O reparo leva segundos; a conferência continua levando uma hora para 254 GiB, e é assim que tem que ser.

Dois casos especiais ganharam uma opção própria, explícita, em vez de uma decisão escondida no script:

- **Arquivos vazios.** Um arquivo de 0 byte não tem conteúdo para provar, então nunca passa no teste da cópia. Eles são apagados com o registro próprio.
- **Arquivos de outra pessoa.** Uma pasta de curso compartilhada, editada por outra conta, deixa versões antigas na pasta de separados quase a cada sync. A versão atual continua na nuvem dessa conta e no meu espelho. Decidi que essas versões podem ser apagadas, sabendo que versões intermediárias se perdem, e isso ficou registrado.

### O que fica na nuvem

Os vídeos brutos do casamento e da lua de mel, cerca de 306 GiB, eram outra opção para sair. Eles ficam. O disco de 8 TB e o backup com restic estão na mesma casa; até existir a cópia *offsite* em rodízio, a nuvem é a única cópia desses arquivos fora de casa. Espaço não é o principal motivo para guardar dados em dois lugares, e arquivo insubstituível é a última coisa a sair da única cópia externa.

A lixeira do Drive também fica cheia por enquanto. O Google só libera o espaço quando a lixeira é esvaziada, então os 326 GiB vão aparecer na conta em uns 30 dias. São os mesmos 30 dias que deixam cada uma dessas remoções reversível pela web.

### Os números

| | Arquivos | GiB |
|---|---:|---:|
| Pastas antigas de cursos de 3D | 1.327 | 254,37 |
| Pasta da empresa de 3D (parte da conta pessoal) | 4.312 | 46,16 |
| Finanças + downloads + vídeo de 2018 | 4.251 | 25,63 |
| **Fora da nuvem** | **9.890** | **326,16** |
| Espelho antes → depois | 37.658 → ≈ 27.770 | 747,7 → ≈ 421,5 |

Erros no sync: 0. Arquivos apagados sem cópia conferida no acervo: 1 arquivo vazio e 3 versões de arquivos de outra conta, os dois por regra explícita. Todo o resto: relido nos dois lados, com uma linha no diário antes de cada exclusão.

## O que isso faz e o que não faz

Garante que nada pessoal sai da nuvem se não houver uma cópia idêntica no disco do acervo, lida e comparada no dia da exclusão.

Não garante que o disco do acervo sobreviva. Isso é trabalho do backup, e é por isso que o próximo passo é uma segunda cópia em outro disco. Também não escolhe entre versões: quando duas versões de um arquivo são diferentes, as duas ficam no acervo, e eu decido depois.
