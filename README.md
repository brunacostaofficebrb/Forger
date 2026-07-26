# Forge 🏋️

PWA pessoal de treino de musculação — acompanha treinos, cargas, medidas e evolução, com contas na nuvem e sincronização entre aparelhos.

**Produção:** https://forge-thiago-personalizado.vercel.app

---

## 📱 Visão de produto

Aplicativo de academia focado em **consistência e progressão**. Cada usuário tem sua própria ficha, histórico e evolução, sincronizados na nuvem.

### Funcionalidades

- **Contas por e-mail/senha** (Supabase Auth) — dados isolados por usuário e sincronizados entre celular/PC. "Lembrar credenciais" e "trocar de conta" na tela de login.
- **Onboarding por plano** — ao criar a conta, escolhe um plano do catálogo (ex.: *Treino do Thiago* ou *Glúteos + Definição*) e todos os treinos daquele plano são vinculados de uma vez. Depois de escolhido, não pergunta de novo.
- **Treinos por usuário + catálogo** — o plano de treino é editável: adicionar do catálogo, criar do zero, editar exercícios e **reordenar por arraste** (⋮⋮, funciona no toque).
- **Sessão de treino ao vivo:**
  - Timer total (tempo real, resiste a travar/recarregar) e **"retomar treino"**.
  - **Timer de descanso automático** ao marcar uma série, com **beep** ao terminar (respeita o toggle "Som do descanso").
  - Campos de carga/reps por série com checkbox "feito", timer de cardio e campo de notas.
- **Histórico** — lista por data + detalhe (carga/reps por série, pendências) e **dicas automáticas** comparando com o treino anterior (▲ subiu / = manteve / ▼ caiu / série pulada) + notas da sessão.
- **Evolução visual** — gráficos (SVG) de **carga por exercício** e de **medidas corporais** (histórico datado), contador de **mesociclo (semana 1–12)** com aviso de reavaliação, e resumo de desempenho.
- **Calendário** — marca qualquer dia com atividade (mesmo sem finalizar o treino) e calcula a **sequência (streak)**.
- **Perfil** — nome/idade/altura editáveis, medições, metas da semana, **tema de cor (Azul/Rosa)**, backup e "apagar tudo".
- **Temas** — Azul (padrão) e Rosa, por usuário e persistido.
- **Backup** — na nuvem (snapshots datados, restauráveis) e local (exportar/importar `.json`).
- **PWA** — instalável e funciona offline (dados salvos localmente e sincronizados quando volta a conexão).

---

## 🛠️ Visão técnica

### Stack
- **Front-end:** HTML + CSS + JavaScript **puro**, tudo em [index.html](index.html) (sem framework, sem build).
- **PWA:** [manifest.json](manifest.json) + service worker [sw.js](sw.js) (cache offline).
- **Back-end:** [Supabase](https://supabase.com) — Postgres + Auth + Row Level Security (RLS). Cliente `@supabase/supabase-js` vendorizado em [vendor/supabase.js](vendor/supabase.js) (para funcionar offline).
- **Hospedagem:** Vercel (site estático; deploy automático a cada push na `main`).

### Modelo de dados (Supabase)
- **`user_state`** — 1 linha por usuário; coluna `data jsonb` guarda o estado inteiro do app. RLS: cada um só acessa a própria linha (`auth.uid() = user_id`).
- **`workout_templates`** — catálogo compartilhado de treinos prontos (leitura para autenticados). Agrupado por `plan` (dono da ficha).
- **`user_backups`** — snapshots datados por usuário (backup na nuvem), RLS por dono.
- **`ping`** — tabela trivial usada pelo keep-alive (leitura liberada).

### Formato do estado (`data` / localStorage `forge_bloom_base_v7`)
```jsonc
{
  "profile":  { "name", "initials", "height", "age" },
  "workouts": [ { "id", "label", "name", "cardio", "cardioLabel",
                  "exercises": [[nome, séries, reps, descanso, dica, buscaYoutube]] } ],
  "history":  [ { "iso", "date", "workout", "workoutId", "detail",
                  "duration", "cardio", "exercises", "notes" } ],
  "lastValues":     { "<workoutId>-<i>": { "kg", "reps" } },
  "measures":       { "weight", "chest", "armR", ... },
  "measureHistory": [ { "iso", "measures" } ],
  "activeDays":     { "<toDateString>": true },
  "goals":    [ ... ], "settings": { "sound", "theme" },
  "mesoStart": "ISO", "current": { ...treino em andamento... }, "updated_at": "ISO"
}
```

### Sincronização (offline-first)
- Todo `save()` grava no `localStorage` e faz **push com debounce** ao Supabase.
- No login, **pull** do estado remoto; resolução por **last-write-wins** via `updated_at`.
- Sem rede, o app opera com o cache local e reenvia ao voltar online.

---

## 🚀 Rodar localmente

Não há build. Basta servir a pasta como estático:

```bash
python3 -m http.server 8000
# abra http://localhost:8000
```

> Login/sync exigem internet (Supabase). Sem chave/rede, o app abre e funciona local.

## 📦 Deploy

Push na `main` → a Vercel publica automaticamente. Arquivos servidos: `index.html`, `sw.js`, `vendor/`, `manifest.json`, `icons/`.

> Ao alterar arquivos, suba a versão do cache em [sw.js](sw.js) (`forge-corrigido-vN`) para forçar atualização offline.

---

## ⚙️ Configuração do Supabase

- **URL/anon key** ficam embutidos no topo do `<script>` em [index.html](index.html) (a chave `anon` é pública e protegida por RLS).
- **Authentication → Providers → Email:** manter **"Confirm email" desligado** (senão o cadastro não retorna sessão e o login não completa).
- Após criar as contas, desligar **"Allow new users to sign up"**.
- **Schema/RLS** e o **seed do catálogo** são aplicados via migrations no projeto.

### Keep-alive (plano free)
Projetos free do Supabase **hibernam após ~7 dias sem atividade**. O workflow [.github/workflows/keepalive.yml](.github/workflows/keepalive.yml) faz uma consulta leve (seg/qua/sex) para manter ativo. Restaurar um projeto pausado é 1 clique ("Resume") no dashboard.

---

## 🔧 Manutenção rápida

- **Editar treinos do catálogo:** tabela `workout_templates` (ou os usuários editam a própria ficha no app).
- **Cores/tema:** variáveis CSS no `:root` (tema azul) e no bloco `:root[data-theme="pink"]` (tema rosa) em [index.html](index.html).
- **Novos campos de estado:** adicionar na migração de estado (topo do script) e no `save()`/render correspondente.

## 📌 Limitações conhecidas
- Sync é *last-write-wins* pelo estado inteiro (adequado a poucos usuários/aparelhos), sem merge campo-a-campo.
- Recuperação de senha não implementada (resetável pelo dashboard do Supabase).
