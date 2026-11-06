# Manual do Pages CMS — Blog Mr. Rizzo

Como escrever, editar e publicar posts do blog pelo navegador.

- **Editor:** https://app.pagescms.org (entrar com a conta GitHub **RizzoAdm**)
- **Repositório:** `RizzoAdm/blog`, branch `main`
- **Blog publicado:** https://rizzoadm.github.io/blog/
- **Configuração do CMS:** `.pages.yml` (raiz do repositório)

---

## 1. O que é o CMS

CMS (*Content Management System*, sistema de gerenciamento de conteúdo) é um painel com formulários para escrever e editar o conteúdo do site sem mexer em arquivos ou código.

O blog é feito de arquivos Markdown (`.md`) guardados no GitHub; o Hugo transforma esses arquivos em páginas. O Pages CMS é uma "fachada" para esses arquivos: você preenche os campos, ele monta o `.md` com o cabeçalho certo (*front matter*) e salva no GitHub.

## 2. O que acontece quando você clica em Save

Tudo é automático — não precisa de `git add`, `git commit` nem `git push`:

1. O CMS grava um **commit** na `main` do GitHub, com a mensagem `Update … (via Pages CMS)`. O commit sai em nome do app Pages CMS (seu e-mail não aparece).
2. O commit dispara o **deploy** (GitHub Actions). Em 1–2 minutos o blog está atualizado.
3. Posts marcados como **rascunho** (*Draft*) nunca são publicados.

Salvar várias vezes o mesmo post gera vários commits e deploys — não tem problema.

## 3. Onde fica cada post

Cada post é uma **pasta** em `content/posts/`, com um arquivo por idioma:

```
content/posts/homelab-vlans/      ← a pasta tem o nome da "chave"
├── index.md                      ← versão em inglês   (lista "Posts (English)")
└── index.pt-br.md                ← versão em português (lista "Artigos (Português)")
```

No menu do CMS:

| Menu | Idioma | Arquivo |
|---|---|---|
| **Posts (English)** | Inglês | `content/posts/<chave>/index.md` |
| **Artigos (Português)** | Português | `content/posts/<chave>/index.pt-br.md` |
| **Media** | — | Imagens em `static/images/` |

## 4. Os campos

Na lista em inglês os nomes dos campos estão em inglês; na lista em português, em português. A lista PT tem um campo a mais: **Slug**.

| Campo | O que é | O que colocar | Exemplo |
|---|---|---|---|
| **Title / Título** | Título no topo do post e nas listas | O título normal, com acentos e pontuação | `VLANs no homelab` |
| **Slug** (só PT) | Final da URL em português | Minúsculas, números e hífens, **sem acento** | `vlans-no-homelab` → `/blog/pt-br/artigos/vlans-no-homelab/` |
| **Key / Chave** | Nome da pasta do post e final da URL em inglês | Minúsculas, números e hífens, sem acento. **A mesma nas duas versões.** Não mude depois de publicar | `homelab-vlans` → `/blog/posts/homelab-vlans/` |
| **Date / Data** | Data do post (ordem nas listas) | Já vem com a data de hoje | `2026-11-06` |
| **Draft / Rascunho** | Marcado = **não** publicado | Deixe marcado enquanto escreve; desmarque e salve para publicar | ☑ |
| **Tags / Etiquetas** | Assuntos do post (viram páginas de filtro) | Uma por linha, em minúsculas. **As mesmas** nas duas versões | `homelab`, `networking` |
| **Summary / Resumo** | Texto curto na lista de posts e na busca | 1–2 frases. Se vazio, o Hugo usa o começo do texto | `How I split IoT devices from my main network with VLANs.` |
| **Comments / Comentários** | Liga/desliga os comentários (giscus) neste post | Deixe marcado. Desmarque só se não quiser comentários | ☑ |
| **Post / Artigo** | O texto do post | Editor visual (negrito, listas, links, imagens) ou Markdown, pelo botão de troca do editor | — |

### Regras da chave e do slug

- Só `a–z`, `0–9` e hífen simples entre palavras: `homelab-vlans` ✅ · `Homelab VLANs` ❌ · `vlans-no-homelab-` ❌ · `configuração` ❌
- Se escrever errado, o CMS não salva e mostra o erro em vermelho embaixo do campo.
- A chave **não pode ser renomeada pelo CMS** (bloqueado de propósito: mudar a chave muda a URL em inglês e separa os comentários).

## 5. Passo a passo: post bilíngue

1. **Posts (English)** → **Add an entry**
   - Title, Key (ex.: `homelab-vlans`), Tags, Summary e o texto.
   - Deixe **Draft** marcado → **Save**.
2. **Artigos (Português)** → **Add an entry**
   - Título em PT, Slug em PT (ex.: `vlans-no-homelab`), **a mesma Chave** (`homelab-vlans`), as mesmas Etiquetas, Resumo e o texto.
   - Deixe **Rascunho** marcado → **Save**.
3. **Revisar:** abra cada versão e releia.
4. **Publicar:** em cada versão, desmarque **Draft / Rascunho** → **Save**. Em 1–2 minutos o post aparece no blog.

Não precisa publicar as duas ao mesmo tempo: dá para publicar o inglês e deixar o português como rascunho (ou o contrário).

## 6. Post em um idioma só

Crie só na lista daquele idioma, com uma chave nova. No blog, o seletor de idioma desse post leva para a página inicial do outro idioma.

Para traduzir depois: crie o post na outra lista usando **a mesma chave**.

## 7. Editar ou despublicar

- **Editar:** abra o post na lista → altere → **Save**.
- **Despublicar sem apagar:** marque **Draft / Rascunho** de novo → **Save**. O post some do blog no próximo deploy.
- **Apagar:** menu **⋯** do post → **Delete**. Num post bilíngue, apague nas duas listas. Os comentários continuam guardados no GitHub Discussions.

## 8. Imagens

- **Antes de enviar fotos, remova os dados EXIF/GPS** — o repositório é público e o CMS não limpa esses dados. No Ubuntu (Terminal, na pasta da foto):

  ```bash
  exiftool -all= -overwrite_original foto.jpg
  ```

  Se o `exiftool` não estiver instalado: `sudo apt install libimage-exiftool-perl`.
- No texto do post, use o botão de imagem do editor e escolha ou envie o arquivo. As imagens ficam em `static/images/` e aparecem no blog como `/blog/images/<arquivo>`.
- Nomes de arquivo são ajustados automaticamente (minúsculas, sem espaços).
- Prefira imagens leves (JPG ou WebP, até ~1600 px de largura).

## 9. Editar também pelo computador (VS Code)

Como o CMS faz commits direto no GitHub, a pasta local fica desatualizada. **Antes de mexer no blog pelo VS Code**, no Terminal:

```bash
cd ~/Projects/blog
git pull
```

Se você só escreve pelo CMS, não precisa fazer nada no computador.

## 10. O que ignorar no CMS

- **Botão `+` em cada linha** (*Add children entry*): cria subpáginas, que o blog não usa.
- **Settings / Configuration**: mostra o `.pages.yml`. Não edite por ali sem necessidade — um erro nesse arquivo tira as listas do menu.

## 11. Problemas comuns

| Problema | Causa provável | Solução |
|---|---|---|
| As duas versões não aparecem ligadas no seletor de idioma | Chaves diferentes | Use a mesma chave nas duas (apague a errada e recrie) |
| Post não aparece no blog | **Draft** marcado, ou deploy ainda rodando | Desmarque Draft e salve; confira em GitHub → `RizzoAdm/blog` → **Actions** |
| Deploy com ❌ vermelho em Actions | Erro no build | Abra o deploy que falhou, copie a mensagem de erro e peça ajuda |
| Não consigo salvar | Campo obrigatório vazio ou com formato inválido | Leia a mensagem em vermelho embaixo do campo |
| `git pull` dá conflito no computador | Arquivo editado no CMS e no VS Code ao mesmo tempo | Não edite o mesmo post nos dois lugares; peça ajuda para resolver o conflito |
| As listas sumiram do menu do CMS | `.pages.yml` com erro | Desfaça a última mudança no `.pages.yml` |
