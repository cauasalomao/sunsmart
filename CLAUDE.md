# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Site do **Sunsmart Hotel** (Pina, Recife/PE) — hotel digital com check-in online, sem recepção física tradicional, sem chaves, posicionado como prático/inteligente/digital. Construído sobre o template Komplexa Hotéis (originalmente preenchido com a fictícia Pousada Vale das Araucárias) e em processo de adaptação para a identidade Sunsmart.

HTML5, CSS3 e JavaScript vanilla — sem build, sem framework, sem `package.json`, sem testes. Português do Brasil em todo o conteúdo.

Este CLAUDE.md foca na arquitetura, no contexto do cliente Sunsmart e nas pegadinhas que não dá pra descobrir só lendo o código. A lista mecânica de termos a trocar (template → cliente) está em `README.md`.

## Desenvolvimento

Sem build e sem instalação:

```bash
python -m http.server 8000
# ou
npx serve .
```

Sem testes e sem linting.

## Estrutura de páginas

Cada página é um diretório com `index.html` para URLs limpas:

```
/                            Home (hero + strip + prévias de sobre/experiências/quartos/blog + CTA)
/sobre/                      Sobre, história
/experiencia/                Atividades / o que faz o hotel ser "smart"
/acomodacoes/                Quartos em grid (sem filtros)
/galeria/                    Galeria completa com lightbox (sem filtros — array único)
/localizacao/                Mapa + atrações próximas
/contato/                    Formulário + mapa
/blog/                       Índice do blog
/blog/_template/             Template para novos posts (marcadores %%PLACEHOLDER%%)
/blog/{slug}/                Posts individuais
```

Raiz também guarda: `hotel-config.json`, `blog-plan.json`, `sitemap.xml`, `robots.txt`, briefings `.md`, `README.md`, `favicon.svg`.

> A estrutura herda de "pousada com chalés/atividades". Para Sunsmart, várias seções (`experiencia`, atividades como cavalgada/pescaria, "atrações próximas" estilo Serra Geral) precisam ser **repropostas** — não só renomeadas. Ex.: `experiencia` vira "Como funciona" / "Tecnologia & autonomia"; "acomodações" passa a listar 3 categorias de quarto (não 4 chalés).

## Arquitetura essencial

### CSS único — `assets/css/style.css`

Todo o estilo em um só arquivo, guiado por custom properties. Os tokens atuais ainda são do template (`--accent: #5b7a3d` verde floresta, `--cta: #f6b230` dourado, `--font-display: 'Pinyon Script'`, `--font-body: 'Raleway'`) — **incompatíveis com a identidade Sunsmart** (digital, urbana, moderna). Ao adaptar, atualize os tokens em um único lugar; o resto da CSS lê deles.

Responsivo em 768/640/480px. Espaçamentos via `clamp()`. Classes curtas quase-BEM: `.rc` room card, `.gi` gallery item, `.exp-card`, `.rp-c`, `.aud-card`, `.fg`, `.btn-gold`/`.btn-green`/`.btn-outline`/`.btn-outline-w`. (Os nomes `btn-gold`/`btn-green` são herdados — pode-se manter como tokens semânticos e só trocar as cores.)

### JS único — `assets/js/main.js`

Toda a interatividade vive aqui. Primitivas principais:

- `sendToWebhook(payload)` — POST JSON para `WEBHOOK_URL` com `{hotel, origem_pagina, url, timestamp, ...payload}`. Usado por todos os forms.
- `pushLead(tipo)` — empurra um evento `gerar_lead` no dataLayer do GTM, com `lead_tipo`.
- `submitContact` — trata o form de `/contato/`.
- Menu mobile (`openMob`/`closeMob`), header sticky (hero-mode ↔ solid no scroll), lightbox (`openLB`/`closeLB`/`navLB`, lê de `LB_SRCS`), banner de cookies, lazy-load observer, swap do título da aba quando a aba fica oculta.

### Constantes no topo de `main.js` (editar aqui, não em valores espalhados)

```js
const WEBHOOK_URL   // URL do webhook n8n/Zapier — todos os forms postam aqui
const HOTEL_NAME    // usado em todos os payloads de webhook → trocar para "Sunsmart Hotel"
const WA_NUMBER     // formato '55 + DDD + número' sem pontuação
const WA_MESSAGE    // texto pré-preenchido em wa.me?text=
const BOOKING_URL   // domínio do site → sunsmarthotel.com.br
const MOTOR_BASE    // base do motor de reservas
```

Formato da URL do motor: `{MOTOR_BASE}/search/{ci}/{co}/{adults}-{age1}-{age2}` (ex.: 2 adultos + crianças de 5 e 8 anos → `.../search/2026-05-10/2026-05-12/2-5-8`). Trocar `MOTOR_BASE` pela URL do motor real (Foco Multimídia ou equivalente) quando o cliente fornecer.

## Dois modais globais injetados via JS

Ambos são inseridos no `<body>` em tempo de execução por IIFEs dentro de `main.js`. **Não adicionar HTML por página para eles.** O HTML dos modais está dentro das IIFEs; o CSS está em `style.css`.

### Modal de captura WhatsApp (classes `.wl-*`)

Intercepta **todo** clique em `a[href*="wa.me/"]` do site (botão flutuante, CTAs do hero, redes sociais no footer etc.). Mostra um card de 340px ancorado bottom-right (desktop) / bottom-center (mobile). Campos obrigatórios: nome/email/telefone. No submit: `pushLead('whatsapp_modal')` → webhook → `form.reset()` → fecha → `window.open('wa.me/{WA_NUMBER}?text=...')`. O botão secundário "📅 Reservar Agora Online" fecha este modal e chama `openBooking()` — não repurpose esse botão para outro destino. Fecha com × / backdrop / Esc.

### Modal de reservas (classes `.bk-*`) — motor Foco Multimídia

Disparado somente por `onclick="openBooking();return false"` explícito nos CTAs "Reservar". **Não intercepta links globalmente** — o trigger é opt-in por botão. Todos os "Reservar"/"Reservar Agora"/"Reservar Estadia"/"Fazer Reserva" seguem esse padrão, incluindo a versão do mobnav que encadeia `closeMob();openBooking();return false`.

Form: check-in / check-out / adultos (1–5) / crianças (0–3). Mudar o select de crianças renderiza N selects de idade (0–12). Submit monta a URL e faz `window.open(url, '_blank', 'noopener')`, depois fecha. O footer do modal tem um fallback por WhatsApp. Fecha com × / backdrop / Esc.

Ao adicionar um novo CTA "Reservar" em qualquer lugar, use:

```html
<a href="#" onclick="openBooking();return false" class="btn-gold">Reservar Agora</a>
```

## Renderização da galeria

`/galeria/index.html` tem um `<div class="gal-g" id="galGrid"></div>` vazio e uma única array inline `GALLERY` (path + alt) no final da página. Um único pass do script monta o grid e popula `LB_SRCS`. Para adicionar/remover fotos, edite só a array — os índices (`openLB(i)`) são calculados.

Para Sunsmart há material real abundante (ver próxima seção). Substitua as 12 entradas placeholder por uma curadoria das fotos reais — provavelmente 30–60 entradas distribuídas pelas categorias visuais do hotel.

## Material fotográfico do Sunsmart

Fotos reais já catalogadas em `assets/img/fotos/`, organizadas em 12 pastas temáticas:

```
01_Fachada_Entrada       (5 fotos)   → hero, sobre, localização
02_Rua_Exterior          (7 fotos)   → contexto urbano, localização
03_Acesso_Recepcao_Virtual (9 fotos) → "Como funciona", check-in digital, totens
04_Lobby_Hall            (7 fotos)   → ambientes comuns
05_Cafe_Coworking        (5 fotos)   → diferencial — espaço de trabalho/café
06_Smart_Market          (5 fotos)   → loja conveniência autoatendimento (diferencial)
07_Cozinha_Comum         (8 fotos)   → cozinha compartilhada
08_Lockers_Roupas        (3 fotos)   → autosserviço de bagagem/roupas
09_CFTV_Seguranca        (1 foto)    → segurança digital
10_Corredores_Sinalizacao (9 fotos)  → ambientação, sinalização moderna
11_Quartos               (20 fotos)  → categorias de quarto
12_Banheiros             (12 fotos)  → banheiros das suítes
```

Cada nome de pasta é a "intenção editorial" da imagem — use isso ao decidir onde colocar cada foto. As fotos das pastas 03/05/06/08/09 são as que **provam o posicionamento "digital/smart"** e devem aparecer com destaque (não enterradas na galeria).

## Padrão de dobras alternadas (página Experiência)

O `<style>` inline em `experiencia/index.html` define modificadores reutilizáveis — vale a pena reaproveitar mesmo após repropor a página para "Como funciona / Tecnologia":

- `.sec-green` / `.sec-green-dark` — seção de cor sólida com texto branco. Os nomes referenciam a paleta antiga; ao trocar para a paleta Sunsmart, mantenha a estrutura e só atualize a cor do `--accent`/`--accent-hover`.
- `.sec-photo` — seção com imagem de fundo fixa. **Não use `background-attachment: fixed`** — está quebrado globalmente por `html { zoom: 0.8 }`. A solução: a seção tem `clip-path: inset(0)` + `isolation: isolate`; `::before` é `position: fixed; inset: 0` com `background-image: var(--bg-photo)`; `::after` é a sobreposição escura, também fixed. O estilo inline define `--bg-photo: url(...)`. Isso dá um parallax real clipado aos limites da seção.
- `.aud-grid` / `.aud-card` — grid de 3 colunas com cards de imagem, overlay em gradiente e label em fonte display no terço inferior.
- `.quad-split` — imagem à esquerda, texto+cards à direita. Combinado com `.quad-cols` (grid 2×2 de itens).

## Placeholders de imagem e logo

Todas as `<img>` e `background-image` ainda apontam para placeholders herdados do template:

- `assets/img/placeholder.svg` — SVG genérico (retângulo + ícone de montanha + texto "FOTO"). O ícone de montanha **não combina** com a identidade Sunsmart; no curto prazo só substitua os caminhos pelas fotos reais (não há mais necessidade do placeholder).
- `assets/img/logo-placeholder.svg` — usado onde o logo do cliente apareceria (header, footer, logo no modal de reservas injetado por JS).
- `favicon.svg` — único favicon referenciado.

**Substituições para Sunsmart**:
1. Trocar `placeholder.svg` pelos paths reais em `assets/img/fotos/{categoria}/IMG_xxxx.jpg` em cada HTML. Os `alt` atuais descrevem chalés/araucárias — reescreva alt descritivo para o que a foto Sunsmart realmente mostra.
2. Substituir `logo-placeholder.svg` pelo logo do Sunsmart quando recebido; se for PNG, atualizar a extensão em cada `<img src=...>` e na constante do modal de reservas em `main.js`.
3. Substituir `favicon.svg` pelo símbolo do Sunsmart.

## Google Maps embed

Os iframes em `/contato/` e `/localizacao/` consultam **pelo nome do negócio**, não pelo endereço. Para Sunsmart, usar:

```
https://maps.google.com/maps?q=Sunsmart+Hotel,+Pina,+Recife+-+PE&output=embed
```

Consultar por endereço faz o Google interpretar segmentos e renderizar uma rota em vez de um pin.

## SEO e structured data

Cada página inclui JSON-LD Schema.org (LodgingBusiness na home, WebPage + BreadcrumbList no resto, BlogPosting nos posts), meta tags Open Graph, Twitter cards, URL canônica. `sitemap.xml` e `robots.txt` na raiz — atualizar `sitemap.xml` sempre que um post ou página for adicionado. Trocar URL canônica para `https://www.sunsmarthotel.com.br/...` ao adaptar.

## Arquivos de configuração

- **`hotel-config.json`** — fonte de verdade do site: contato (telefone/e-mail/WA), endereço + coordenadas, categorias de quarto, diferenciais, pacotes, atrações próximas, integrações (`webhook_url`, `booking_engine_url`), design tokens, configurações de blog. Mantenha em sincronia com as constantes do `main.js` quando valores mudarem. **Comece por este arquivo ao adaptar o site para Sunsmart** — depois propague as mudanças para HTML/JS/CSS.
- **`blog-plan.json`** — estratégia editorial, regras de SEO, spec do template de post, lista `published` e fila `upcoming`. Os pilares atuais (Destino/Experiência/Família/Dicas) servem para pousada de serra; **repensar pilares** para Sunsmart (ex.: "Recife para o viajante a trabalho", "Como funciona o check-in digital", "Pina e arredores", "Compras/eventos/saúde em Recife").

## Fluxo de criação de post no blog

1. Pegar próximo item em `blog-plan.json` → `upcoming`.
2. Copiar `blog/_template/index.html` → `blog/{slug}/index.html`.
3. Trocar cada marcador `%%PLACEHOLDER%%` (título, meta desc, slug, data, seções de conteúdo, palavra-chave).
4. Escrever 800–1200 palavras: intro com keyword, 3–5 `<h2>`, 2+ links internos, `.blog-cta-box` no fim.
5. Adicionar card em `blog/index.html` dentro de `#blogGrid`.
6. Adicionar `<url>` em `sitemap.xml`.
7. Mover item de `upcoming` para `published` em `blog-plan.json`.
8. Commitar localmente (push só com link + ordem explícitos — ver "Preferência do usuário").

### Checklist de SEO por post

`<title>` único com keyword (formato `{Título} | Blog Sunsmart Hotel`), meta description ≤155 caracteres, URL canônica, Open Graph, `article:published_time`, JSON-LD `BlogPosting` + `BreadcrumbList`, `<h1>` único, `<h2>` por seção, ≥2 links internos, `.blog-cta-box` no fim.

## Contexto do hotel Sunsmart (decisões de conteúdo vêm daqui)

Fonte: `# Briefing do Cliente.md` + `# Sunsmart Hotel - Briefing do Sistema.md`.

- **Identidade:** "Prático. Inteligente. Digital." Hotel digital nascido na pandemia, inspirado em redes econômicas tipo Ibis Budget mas mais enxuto e tecnológico. Não há equivalente direto no mercado local de Recife percebido pelos fundadores.
- **Localização:** Bairro do Pina, Recife/PE. Próximo a Shopping RioMar (10 min a pé), Praia do Pina (<1 km), Recife Antigo, Polo Médico, Consulado Americano, Centro de Convenções de Pernambuco, Aeroporto Internacional do Guararapes (15 min).
- **Estrutura:** 20 suítes — Duplo c/ cama de casal, Duplo c/ 2 solteiros, Triplo c/ 3 solteiros. Quartos ~10–12m². Áreas comuns: lobby, cozinha compartilhada, café/coworking, smart market (autoatendimento), lockers.
- **Operação:** check-in 100% online, sem chaves físicas (senha enviada por e-mail/WhatsApp após pagamento), sem recepção 24h presencial — equipe reduzida, camareiras presentes em horário comercial. Atendimento por canais digitais.
- **Público:**
  - Corporativo, casais sem filhos, jovens, viajantes "smartphone-natives".
  - Estadias curtas: consultas médicas (Polo Médico), processo de visto americano (Consulado), eventos no Centro de Convenções/Recife Expo, compras no RioMar, reuniões nos empresariais da região.
  - Origem majoritária: Nordeste.
- **Diferenciais:** localização estratégica · check-in sem fila · operação digital autônoma · custo-benefício · modelo inovador · café/coworking · smart market · cozinha compartilhada · acesso 24h livre.
- **Restrições:** aceita crianças e bebês (categoria duplo casal); proíbe fumar, animais, eventos. Check-in 14h–23h, checkout 10h–11h. Quarto individual de solteiro **não** aceita crianças.
- **Sazonalidade:** Carnaval e Réveillon → 100% ocupação, tarifas até 4× a média.
- **Promoções ativas:** cupons para recorrentes, parcerias comerciais.
- **Objetivos comerciais:** aumentar reservas diretas, atrair público digital maduro, reforçar jornada autônoma, reduzir atendimento humano.
- **Tom de voz:** moderno, direto, eficiente, urbano. Foca em **autonomia, agilidade, tecnologia, custo-benefício**. Evita linguagem afetuosa/familiar/rural — é o oposto do tom de "pousada acolhedora" do template original.
- **Contraponto importante:** o template foi escrito para uma pousada serrana familiar. Toda decisão de copy/design herdada (vocabulário "acolhedor", "como receber amigos em casa", paleta verde floresta, fonte caligráfica Pinyon Script) precisa ser repensada para Sunsmart, não só traduzida.
- **Idioma:** português brasileiro.
- **Responsável pelo briefing:** Adalberto, Proprietário.

## Convenções

- Dispatcher de webhook trata o roteamento de leads; todo form passa por `sendToWebhook` + `pushLead`.
- Modais usam o toggle da classe `.open`. `document.body.style.overflow = 'hidden'` enquanto aberto, restaurado no fechamento.
- Forms `preventDefault` → webhook → evento GTM → ação de UI (redirect / WhatsApp / estado de sucesso).
- `<style>` inline em subpáginas (sobre, experiencia) guardam regras específicas da página; o global fica em `assets/css/style.css`.
- O path do logo no markup injetado por JS usa `/assets/img/logo-placeholder.svg` absoluto para resolver corretamente a partir de qualquer profundidade.
- Não edite a CSS legada `.wa-modal` — é herança do template base sem uso; o modal de WhatsApp vivo usa classes `.wl-*`.

## Preferência do usuário — política de Git

Commits locais automáticos na branch `main` são ok e esperados após cada alteração de código, sem precisar pedir.

**Push para o GitHub, porém, NÃO é automático.** Um push anterior do template sobrescreveu o repositório de um cliente real em produção. Para evitar reincidência:

- **Nunca** rodar `git push`, `gh pr create`, `gh repo create`, ou qualquer comando que envie conteúdo deste diretório para o GitHub por iniciativa própria.
- Publicar só quando o usuário **explicitamente** fornecer, na mesma instrução, **(a)** a URL do repositório remoto correto **e (b)** a ordem clara de dar push/publicar. Ambos os itens precisam estar presentes — um sem o outro não basta.
- Antes de executar o push autorizado, rodar `git remote -v` e confirmar que o remote aponta para a URL fornecida (adicionar/ajustar `origin` se necessário).
