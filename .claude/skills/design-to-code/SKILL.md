---
name: design-to-code
description: "Design to Code — Figma → NMS. Converte designs do Figma em codigo production-ready para o projeto GE Vernova NMS com validacao visual via Playwright."
disable-model-invocation: true
argument-hint: "[URLs do Figma opcionais]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Agent
  - mcp__figma__get_design_context
  - mcp__figma__get_screenshot
  - mcp__figma__get_metadata
---

# Design to Code — Figma → NMS

Voce e um especialista em converter designs do Figma em codigo production-ready para o projeto GE Vernova NMS. Voce utiliza o Figma MCP server para ler designs e o Playwright para validar visualmente o resultado.

**IMPORTANTE:** Siga as regras da secao "Figma MCP — Design-to-Code Rules" do `CLAUDE.md` deste projeto para todas as decisoes de conversao. Nao duplique essas regras — consulte-as diretamente.

---

## Fase 0: Verificacao de pre-requisitos

Antes de iniciar, verifique que o ambiente esta pronto. **Nao prossiga ate que todos os pre-requisitos estejam confirmados.**

### 1. Figma MCP server

Verifique se as ferramentas do Figma MCP estao acessiveis tentando listar as ferramentas disponiveis. Voce precisa de acesso a:
- `get_design_context`
- `get_screenshot`
- `get_metadata`

Se nao estiverem disponiveis, informe ao usuario que o Figma MCP server precisa estar configurado e conectado.

### 2. Docker Compose

Execute `docker compose ps` no diretorio `frontend/` para verificar se o container `frontend` esta rodando.

- Se **nao estiver rodando**, oriente o usuario:
  ```
  cd frontend && docker compose up -d
  ```
  Aguarde o container ficar healthy antes de prosseguir.

- O container Playwright usa `profiles: ["test"]` e sera invocado sob demanda — nao precisa estar rodando previamente.

---

## Fase 1: Coleta interativa

### Passo 1: Quantidade

Pergunte ao usuario:
> Quantas telas ou componentes voce deseja converter?

### Passo 2: Para cada tela, colete as informacoes

Para cada tela, pergunte **na seguinte ordem**:

1. **"Esta tela tem variantes dark e light?"**
   - Se **sim** → solicite 2 URLs do Figma (uma para dark, uma para light)
   - Se **nao** → solicite 1 URL do Figma

   > **IMPORTANTE:** A pergunta dark/light serve para determinar o **modo de referencia visual** para validacao (quantas URLs coletar e contra quais screenshots validar). O resultado final sera **sempre um unico componente** com estilos via CSS variables e classe `dark` (next-themes) — **nunca** implementacoes separadas por tema.

2. **"Qual e a rota de destino na aplicacao?"**
   - Exemplos: `/login`, `/alarms`, `/connections`, `/config`
   - Esta informacao e usada para inferir o path dos arquivos no projeto

Ao final da coleta, apresente um resumo de todas as telas coletadas e peca confirmacao antes de prosseguir:

```
Resumo:
- Tela 1: Login (dark/light)
  - Dark: https://figma.com/design/...?node-id=28-162
  - Light: https://figma.com/design/...?node-id=28-285
  - Rota: /login
```

---

## Fase 2: Leitura do design

Processe **uma tela por vez**. Para cada tela:

### Passo 1: Extrair fileKey e nodeId

Parse a URL do Figma para extrair:
- **fileKey**: segmento apos `/design/`
- **nodeId**: valor do parametro `node-id` (converter `-` para `:`)

Exemplo:
- URL: `https://www.figma.com/design/UHjDKKBSHABDxa79E0NmP5/Handoff?node-id=28-162`
- fileKey: `UHjDKKBSHABDxa79E0NmP5`
- nodeId: `28:162`

### Passo 2: Buscar contexto do design

Chame `get_design_context(fileKey, nodeId)` para obter:
- Estrutura de layout (Auto Layout, constraints, sizing)
- Tipografia (fonte, tamanho, peso)
- Cores e design tokens
- Estrutura de componentes
- Espacamento e padding

**Fallback em cascata:**
1. Se a resposta for muito grande ou truncada → chame `get_metadata(fileKey, nodeId)` para obter o mapa de nos
2. Identifique os nos filhos necessarios do metadata
3. Faca re-fetch individual com `get_design_context(fileKey, childNodeId)` para cada no
4. Se ainda falhar → informe o erro ao usuario e peca outra URL

### Passo 3: Capturar referencia visual

Chame `get_screenshot(fileKey, nodeId)` para obter o screenshot de referencia.

Se a tela tem variantes dark/light, repita os passos 1-3 para **ambas as URLs** e analise as diferencas de estilo entre os modos (cores, backgrounds, borders, shadows).

### Passo 4: Apresentar compreensao

Apresente ao usuario sua compreensao do design:
- O que a tela/componente representa
- Quais elementos visuais contem (formularios, botoes, imagens, icones, etc.)
- Se dark/light: quais tokens/estilos variam entre os modos
- Qual sera a estrategia de conversao (quais componentes reutilizar, quais criar)

**PARE e aguarde confirmacao do usuario antes de prosseguir para a conversao.**

---

## Fase 3: Conversao

### Passo 0 (OBRIGATORIO): Scan de componentes existentes

**Antes de criar qualquer componente novo**, escaneie os diretorios:
- `frontend/components/ui/` — primitivos shadcn/ui
- `frontend/components/layout/` — Sidebar, Header, Drawer, ModeToggle

Liste os componentes existentes que podem ser reutilizados para esta tela. **So crie novos componentes se nao houver equivalente.**

### Passo 1: Inferir paths de arquivos

Baseado na rota informada e nas convencoes do projeto:

| Rota | Path do arquivo |
|------|----------------|
| `/login` | `app/login/page.tsx` |
| `/` (topologia) | `app/(dashboard)/page.tsx` |
| `/<feature>` | `app/(dashboard)/<feature>/page.tsx` |
| Componente UI novo | `components/ui/<nome>.tsx` |
| Componente de layout | `components/layout/<nome>.tsx` |

### Passo 2: Gerar codigo

Siga as regras da secao "Figma MCP — Design-to-Code Rules" do `CLAUDE.md`. Em resumo:

- **Tokens CSS:** Use CSS variables de `app/globals.css` — nunca hex hardcoded
- **Icones:** Use apenas `@phosphor-icons/react` — nunca Lucide/Heroicons
- **Componentes:** Padrao shadcn/ui (CVA, `cn()`, `React.forwardRef`, `displayName`)
- **Textos:** Todos via `next-intl` — nunca strings hardcoded. Adicione chaves nos 4 locales (`messages/{en,pt-BR,es,fr}.json`)
- **Dark/light:** Componente unico. Estilos variam via CSS variables e classe `dark` (next-themes). Tokens semanticos (`--background`, `--foreground`, `--primary`, etc.) fazem auto-switch.
- **Path alias:** Use `@/components/ui/button` (nunca caminhos relativos)
- **Assets:** Se o Figma MCP retornar URL localhost para imagem/SVG, use diretamente. Baixe assets estaticos para `public/`

### Passo 3: Criar/editar arquivos

- Crie ou edite os arquivos identificados no Passo 1
- Baixe quaisquer assets (imagens, SVGs) retornados pelo Figma MCP para `public/`
- Adicione chaves de traducao nos 4 arquivos de locale:
  - `frontend/messages/en.json`
  - `frontend/messages/pt-BR.json`
  - `frontend/messages/es.json`
  - `frontend/messages/fr.json`

---

## Fase 4: Validacao visual

### Passo 1: Aguardar hot reload

Apos criar/editar os arquivos, aguarde alguns segundos para o hot reload do Next.js processar as mudancas.

### Passo 2: Capturar screenshot do resultado

Use o Playwright **via Docker** para capturar um screenshot da rota renderizada:

```bash
cd frontend && docker compose run --rm playwright node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext({
    viewport: { width: 1440, height: 900 },
    colorScheme: 'light'
  });
  const page = await context.newPage();
  await page.goto('http://frontend:3000<ROTA>');
  await page.waitForLoadState('networkidle');
  await page.screenshot({ path: '/tmp/screenshot-light.png', fullPage: true });
  await browser.close();
})();
"
```

Se a tela tem variante dark, capture tambem com `colorScheme: 'dark'`:

```bash
cd frontend && docker compose run --rm playwright node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext({
    viewport: { width: 1440, height: 900 },
    colorScheme: 'dark'
  });
  const page = await context.newPage();
  await page.goto('http://frontend:3000<ROTA>');
  await page.waitForLoadState('networkidle');
  await page.screenshot({ path: '/tmp/screenshot-dark.png', fullPage: true });
  await browser.close();
})();
"
```

### Passo 3: Comparar visualmente

Leia os screenshots capturados pelo Playwright e compare com os screenshots do Figma obtidos na Fase 2.

**Criterios de comparacao (qualitativa, nao pixel-diff):**
- Layout: posicao, alinhamento e tamanho dos elementos
- Tipografia: fonte, tamanho, peso, line-height
- Cores: background, foreground, borders, shadows
- Espacamento: padding, margin, gaps
- Componentes: todos os elementos presentes e corretos
- Dark mode (se aplicavel): tokens corretos aplicados

**IMPORTANTE:** Esta e uma comparacao qualitativa. Diferencas sutis de anti-aliasing e renderizacao de fontes entre Figma e browser sao esperadas e devem ser ignoradas. Foque em desvios significativos de layout, cor e estrutura.

### Passo 4: Auto-correcao (ate 5 tentativas)

Se identificar desvios significativos:

1. Descreva os desvios encontrados
2. Corrija o codigo
3. Aguarde hot reload
4. Capture novo screenshot
5. Compare novamente

**Limite: 5 tentativas.** Apos 5 tentativas sem convergencia:
- Apresente o resultado atual ao usuario
- Liste os desvios remanescentes
- Deixe o usuario decidir se aceita ou quer ajustar manualmente

### Passo 5: Proxima tela

Se houver mais telas na fila, volte a **Fase 2** para a proxima tela.

---

## Finalizacao

Apos completar todas as telas, apresente ao usuario:

### Resumo do que foi feito
- Arquivos criados/editados
- Componentes reutilizados vs criados
- Assets baixados
- Chaves de traducao adicionadas

### Itens fora do escopo

**IMPORTANTE:** Alerte o usuario que os seguintes itens do checklist de nova feature **nao sao cobertos** por esta skill e devem ser completados manualmente:

- [ ] Adicionar item de navegacao na sidebar (`components/layout/sidebar.tsx`) — se aplicavel
- [ ] Adicionar rota no matcher do middleware (`middleware.ts`) — se necessario
- [ ] Adicionar JSON stub em `docker/nginx/data/` — se nova API necessaria
- [ ] Adicionar rota no `docker/nginx/nginx.conf` — se nova API necessaria
- [ ] Escrever testes E2E relevantes em `e2e/`
- [ ] Verificar build: `docker compose exec frontend sh -c "npm run build"`
