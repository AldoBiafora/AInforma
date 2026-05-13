# AInforma — Piano di progetto

App mobile di informazione sull'intelligenza artificiale rivolta a sviluppatori, tecnici, curiosi e appassionati. Stack: React Native + Expo + Supabase.

---

## Obiettivo

Aggregare e presentare contenuti AI aggiornati (news, tool, paper, tutorial, podcast/video) con frequenza giornaliera, settimanale e mensile. L'app deve funzionare per due profili: chi vuole capire cosa succede nel mondo AI (appassionati) e chi vuole approfondimenti tecnici (sviluppatori).

---

## Stack tecnico

| Layer | Tecnologia |
|---|---|
| Framework mobile | React Native + Expo SDK 51+ |
| Navigation | Expo Router (file-based) |
| Backend / DB | Supabase (PostgreSQL) |
| Auth | Supabase Auth (email + Google OAuth) |
| Storage | Supabase Storage (immagini, thumbnail) |
| API esterne | RSS feeds, arXiv API, YouTube Data API v3 |
| Aggregazione contenuti | Supabase Edge Functions (Deno) |
| Notifiche push | Expo Notifications + Supabase triggers |
| State management | Zustand |
| Fetch / cache | TanStack Query (React Query) |
| UI components | Custom + NativeWind (Tailwind per RN) |
| Testing | Jest + React Native Testing Library |

---

## Struttura delle schermate (Expo Router)

```
app/
├── (tabs)/
│   ├── index.tsx          # Feed principale
│   ├── tools.tsx          # Directory tool AI
│   ├── learn.tsx          # Tutorial + guide
│   ├── media.tsx          # Podcast e video
│   └── profile.tsx        # Profilo utente + preferenze
├── article/[id].tsx       # Dettaglio articolo
├── tool/[id].tsx          # Dettaglio tool
├── paper/[id].tsx         # Dettaglio paper arXiv
├── search.tsx             # Ricerca globale
├── onboarding.tsx         # Wizard onboarding (primo avvio)
└── auth/
    ├── login.tsx
    └── register.tsx
```

---

## Schema database Supabase

### Tabelle principali

```sql
-- Articoli / news
create table articles (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  summary text,
  content text,
  url text not null,
  source text not null,           -- "OpenAI", "Google DeepMind", ecc.
  source_type text not null,      -- 'news' | 'research' | 'blog'
  difficulty text default 'all',  -- 'all' | 'technical' | 'beginner'
  tags text[] default '{}',
  published_at timestamptz not null,
  created_at timestamptz default now(),
  thumbnail_url text,
  read_count int default 0,
  is_featured boolean default false
);

-- Tool AI
create table tools (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  url text not null,
  category text not null,         -- 'llm' | 'image' | 'audio' | 'code' | 'productivity' | 'research'
  tags text[] default '{}',
  pricing text default 'free',    -- 'free' | 'freemium' | 'paid'
  is_new boolean default false,
  added_at timestamptz default now(),
  logo_url text,
  upvote_count int default 0
);

-- Paper arXiv
create table papers (
  id uuid primary key default gen_random_uuid(),
  arxiv_id text unique not null,
  title text not null,
  abstract text,
  authors text[] default '{}',
  categories text[] default '{}',
  pdf_url text,
  published_at timestamptz,
  fetched_at timestamptz default now(),
  simplified_summary text          -- generato da Edge Function con Claude API
);

-- Tutorial / guide
create table tutorials (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text,
  url text,
  content_type text default 'article', -- 'article' | 'video' | 'course'
  difficulty text default 'beginner',  -- 'beginner' | 'intermediate' | 'advanced'
  duration_minutes int,
  tags text[] default '{}',
  published_at timestamptz,
  thumbnail_url text
);

-- Media (podcast + video)
create table media_items (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  description text,
  url text not null,
  media_type text not null,       -- 'podcast' | 'video'
  source text,                    -- nome canale / podcast
  duration_seconds int,
  thumbnail_url text,
  published_at timestamptz,
  tags text[] default '{}'
);

-- Profili utente
create table profiles (
  id uuid primary key references auth.users(id),
  username text,
  avatar_url text,
  preferred_difficulty text default 'all',
  preferred_tags text[] default '{}',
  notification_daily boolean default true,
  notification_weekly boolean default true,
  created_at timestamptz default now()
);

-- Segnalibri utente
create table bookmarks (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  content_type text not null,     -- 'article' | 'tool' | 'paper' | 'tutorial' | 'media'
  content_id uuid not null,
  created_at timestamptz default now(),
  unique(user_id, content_type, content_id)
);

-- Cronologia lettura
create table reading_history (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  content_type text not null,
  content_id uuid not null,
  read_at timestamptz default now()
);
```

### Row Level Security

```sql
-- Tutti leggono i contenuti pubblici
alter table articles enable row level security;
create policy "public read" on articles for select using (true);

-- Solo l'utente vede i propri segnalibri
alter table bookmarks enable row level security;
create policy "own bookmarks" on bookmarks
  using (auth.uid() = user_id);

-- Solo l'utente vede la propria cronologia
alter table reading_history enable row level security;
create policy "own history" on reading_history
  using (auth.uid() = user_id);
```

---

## Edge Functions Supabase

### 1. `fetch-articles` — aggregatore RSS giornaliero

Eseguita ogni giorno alle 07:00 via cron Supabase.

```
Sorgenti RSS da aggregare:
- The Verge AI: https://www.theverge.com/ai-artificial-intelligence/rss/index.xml
- MIT Technology Review: https://www.technologyreview.com/feed/
- VentureBeat AI: https://venturebeat.com/category/ai/feed/
- Ars Technica AI: https://arstechnica.com/tag/artificial-intelligence/feed/
- Interconnects (newsletter tecnica): https://www.interconnects.ai/feed
- Ahead of AI: https://magazine.sebastianraschka.com/feed

Logica:
1. Fetch tutti i feed RSS
2. Parse XML → oggetti strutturati
3. Dedup su URL (upsert con on conflict do nothing)
4. Inserimento in tabella articles
5. Per ogni articolo nuovo: trigger notifica push agli utenti con notification_daily = true
```

### 2. `fetch-papers` — paper arXiv settimanali

Eseguita ogni lunedì alle 08:00.

```
Categorie arXiv da monitorare:
- cs.AI   (Artificial Intelligence)
- cs.LG   (Machine Learning)
- cs.CL   (Computation and Language / NLP)
- cs.CV   (Computer Vision)
- stat.ML (Statistical Machine Learning)

Logica:
1. Query API arXiv: https://export.arxiv.org/api/query?search_query=cat:cs.AI&start=0&max_results=50
2. Filtra paper degli ultimi 7 giorni
3. Per ogni paper: chiama Claude API (claude-haiku) per generare simplified_summary (max 150 parole, linguaggio accessibile)
4. Inserimento in tabella papers
```

### 3. `fetch-tools` — nuovi tool AI

Eseguita ogni venerdì alle 09:00.

```
Sorgenti:
- Product Hunt API (categoria AI Tools)
- There's An AI For That API
- Futurepedia RSS

Logica:
1. Fetch tool delle ultime 7 giorni
2. Classifica per categoria (llm / image / audio / code / productivity / research)
3. Setta is_new = true per tool aggiunti questa settimana
4. Rimuovi is_new dai tool più vecchi di 14 giorni
```

### 4. `generate-digest` — digest settimanale/mensile

Eseguita ogni domenica alle 10:00 (settimanale) e il 1° del mese (mensile).

```
Logica:
1. Raccoglie top articoli per read_count e engagement della settimana/mese
2. Raccoglie top 5 paper per area
3. Raccoglie top 10 tool nuovi
4. Genera testo digest con Claude API
5. Salva digest in tabella dedicata o invia via notifica push
```

---

## Sezioni dell'app — dettaglio funzionale

### 1. Feed principale (`/`)

Aggregatore cronologico di tutti i contenuti con filtri per:
- **Frequenza**: Oggi / Questa settimana / Questo mese
- **Tipo**: News / Paper / Tool / Tutorial / Video
- **Livello**: Tutti / Tecnico / Beginner
- **Tag**: LLM / Vision / Audio / Robotica / Policy / Open Source / ecc.

Card news mostra: fonte + badge livello + titolo + summary (2 righe) + tag + data + pulsante segnalibro.

Sezione "In evidenza" in cima: 3 card orizzontali scrollabili con i contenuti più letti del giorno.

### 2. Tool AI (`/tools`)

Directory navigabile e ricercabile di tool AI.

Filtri: categoria (LLM / Image / Audio / Code / Productivity / Research) + pricing (Free / Freemium / Paid) + ordinamento (Più nuovi / Più votati).

Card tool: logo + nome + descrizione breve + categoria + badge pricing + badge "Nuovo" se aggiunto negli ultimi 14 giorni + upvote button.

Dettaglio tool (`/tool/[id]`): descrizione estesa + link + categoria + tag + data aggiunta + sezione "Tool simili".

### 3. Impara (`/learn`)

Tutorial e guide organizzati per livello (Beginner / Intermediate / Advanced) e argomento.

Sottosezioni:
- **Concetti base**: spiegazioni accessibili di LLM, diffusion model, RAG, ecc.
- **Guide pratiche**: come usare strumenti specifici, prompt engineering, fine-tuning
- **Paper spiegati**: versione semplificata dei paper arXiv (generata da Edge Function)
- **Roadmap**: percorsi consigliati per imparare ML / NLP / Computer Vision

### 4. Media (`/media`)

Aggregatore di podcast e video AI con tabs Podcast / Video.

Per i podcast: titolo episodio + podcast di origine + durata + descrizione + link player esterno.

Per i video: thumbnail + titolo + canale + durata + link YouTube.

Sorgenti video da monitorare:
- Andrej Karpathy (YouTube)
- Yannic Kilcher (YouTube)
- AI Explained (YouTube)
- Two Minute Papers (YouTube)
- Lex Fridman (episodi AI)

Sorgenti podcast:
- Latent Space
- The TWIML AI Podcast
- Practical AI
- Cognitive Revolution
- No Priors

### 5. Profilo (`/profile`)

- Preferenze di difficoltà (Tutti / Tecnico / Beginner)
- Tag di interesse selezionabili
- Gestione notifiche (giornaliera / settimanale)
- Cronologia lettura
- Segnalibri salvati
- Statistiche personali (articoli letti, tool esplorati, paper salvati)

### 6. Ricerca (`/search`)

Full-text search su tutti i contenuti tramite Supabase `to_tsvector` / `plainto_tsquery`.

Risultati raggruppati per tipo con anteprima.

### 7. Onboarding (`/onboarding`)

3 schermate al primo avvio:
1. Benvenuto + pitch dell'app
2. Selezione livello (Curioso / Appassionato / Sviluppatore)
3. Selezione tag di interesse (LLM, Vision, Audio, Robotica, Policy, Open Source, Startup AI)

Salva preferenze in `profiles` e personalizza il feed da subito.

---

## Funzionalità trasversali

### Digest settimanale e mensile

Sezione dedicata nel feed con card speciali "Recap settimana" e "Recap mese". Contiene:
- Top 5 news della settimana/mese
- Top 3 paper
- Tool più discussi
- Una frase/citazione dal mondo AI

### Notifiche push

- Notifica giornaliera (ora configurabile) con i titoli top del giorno
- Notifica settimanale con il digest
- Notifica immediata per news di rottura (es. rilascio nuovo modello major)

### Modalità offline

Cache locale con TanStack Query dei contenuti letti di recente. Articoli completi salvati localmente se segnalibrati.

### "Spiega in semplice"

Su ogni articolo tecnico o paper: pulsante "Spiega in semplice" che chiama Claude API (claude-haiku) e genera una spiegazione accessibile in italiano. Risultato cachato in Supabase per non ripetere la chiamata.

---

## Struttura cartelle progetto

```
ainforma/
├── app/                          # Expo Router screens
│   ├── (tabs)/
│   │   ├── index.tsx
│   │   ├── tools.tsx
│   │   ├── learn.tsx
│   │   ├── media.tsx
│   │   └── profile.tsx
│   ├── article/[id].tsx
│   ├── tool/[id].tsx
│   ├── paper/[id].tsx
│   ├── search.tsx
│   ├── onboarding.tsx
│   └── auth/
│       ├── login.tsx
│       └── register.tsx
├── components/
│   ├── cards/
│   │   ├── ArticleCard.tsx
│   │   ├── ToolCard.tsx
│   │   ├── PaperCard.tsx
│   │   ├── MediaCard.tsx
│   │   └── DigestCard.tsx
│   ├── feed/
│   │   ├── FeedFilter.tsx
│   │   ├── FeedHeader.tsx
│   │   └── FeaturedRow.tsx
│   ├── ui/
│   │   ├── Badge.tsx
│   │   ├── Chip.tsx
│   │   ├── BookmarkButton.tsx
│   │   ├── DifficultyBadge.tsx
│   │   └── EmptyState.tsx
│   └── layout/
│       ├── ScreenWrapper.tsx
│       └── SectionHeader.tsx
├── hooks/
│   ├── useArticles.ts
│   ├── useTools.ts
│   ├── usePapers.ts
│   ├── useBookmarks.ts
│   ├── useProfile.ts
│   └── useSearch.ts
├── lib/
│   ├── supabase.ts               # client Supabase
│   ├── queryClient.ts            # TanStack Query config
│   └── notifications.ts          # Expo Notifications setup
├── stores/
│   ├── authStore.ts              # Zustand: utente autenticato
│   └── feedStore.ts             # Zustand: filtri feed attivi
├── supabase/
│   ├── migrations/               # SQL migrations
│   └── functions/
│       ├── fetch-articles/       # Edge Function
│       ├── fetch-papers/
│       ├── fetch-tools/
│       └── generate-digest/
├── types/
│   └── index.ts                  # tipi TypeScript condivisi
├── constants/
│   ├── tags.ts                   # lista tag disponibili
│   ├── sources.ts                # lista sorgenti RSS
│   └── categories.ts             # categorie tool
└── assets/
    └── images/
```

---

## Fasi di sviluppo

### Fase 1 — Fondamenta (settimana 1–2)

- [ ] Setup progetto Expo con Expo Router
- [ ] Configurazione Supabase: progetto, schema DB, RLS
- [ ] Auth: login / registrazione / logout con Supabase Auth
- [ ] Client Supabase + TanStack Query setup
- [ ] Zustand store per auth e filtri
- [ ] Schermata onboarding (3 step)
- [ ] Struttura tab navigator base

### Fase 2 — Feed e contenuti (settimana 3–4)

- [ ] Edge Function `fetch-articles` con 3+ sorgenti RSS
- [ ] Schermata Feed con lista articoli
- [ ] FeedFilter (frequenza + tipo + livello)
- [ ] ArticleCard component
- [ ] Dettaglio articolo (`/article/[id]`)
- [ ] Segnalibri: salva / rimuovi (tabella bookmarks)
- [ ] Cronologia lettura (tabella reading_history)

### Fase 3 — Tool e Paper (settimana 5–6)

- [ ] Edge Function `fetch-papers` con arXiv API
- [ ] Edge Function `fetch-tools` con Product Hunt API
- [ ] Schermata Tool con filtri e griglia
- [ ] Dettaglio tool
- [ ] Schermata Paper con lista
- [ ] Dettaglio paper con `simplified_summary`
- [ ] Feature "Spiega in semplice" (Claude API via Edge Function)

### Fase 4 — Learn e Media (settimana 7–8)

- [ ] Schermata Learn con sezioni Concetti / Guide / Roadmap
- [ ] Schermata Media con tab Podcast / Video
- [ ] Edge Function o seed manuale per contenuti Learn iniziali
- [ ] Player esterno per podcast (link + descrizione)
- [ ] Integrazione YouTube Data API per video

### Fase 5 — Digest e Notifiche (settimana 9–10)

- [ ] Edge Function `generate-digest` settimanale e mensile
- [ ] DigestCard nel feed
- [ ] Setup Expo Notifications
- [ ] Notifica giornaliera configurabile
- [ ] Notifica settimanale con digest
- [ ] Preferenze notifiche in Profile

### Fase 6 — Ricerca e Profilo (settimana 11–12)

- [ ] Full-text search Supabase su tutti i contenuti
- [ ] Schermata Search con risultati per tipo
- [ ] Schermata Profile completa
- [ ] Statistiche utente (articoli letti, tool visti, paper salvati)
- [ ] Sezione Segnalibri nel profilo

### Fase 7 — Rifinitura e rilascio (settimana 13–14)

- [ ] Modalità offline con cache TanStack Query
- [ ] Ottimizzazione performance (FlatList, lazy loading immagini)
- [ ] Gestione errori e stati vuoti (EmptyState)
- [ ] Dark mode
- [ ] Test Jest per hook e componenti critici
- [ ] Build Expo per iOS e Android
- [ ] Submission App Store / Google Play

---

## Comandi utili

```bash
# Setup progetto
npx create-expo-app ainforma --template tabs
cd ainforma
npx expo install expo-router expo-notifications
npm install @supabase/supabase-js @tanstack/react-query zustand
npm install nativewind tailwindcss
npm install react-native-safe-area-context react-native-screens

# Sviluppo
npx expo start

# Supabase CLI
npx supabase init
npx supabase start                        # locale
npx supabase db push                      # applica migrations
npx supabase functions deploy fetch-articles

# Test
npm test
npm test -- --coverage

# Build
npx expo build:ios
npx expo build:android
```

---

## Variabili d'ambiente

```env
EXPO_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key   # solo Edge Functions
ANTHROPIC_API_KEY=your-anthropic-key              # solo Edge Functions
YOUTUBE_API_KEY=your-youtube-key                  # solo Edge Functions
```

---

## Note per Claude Code

- Usa sempre TypeScript strict mode
- Ogni componente in file separato con named export
- Hook custom per ogni entità Supabase (useArticles, useTools, ecc.)
- TanStack Query per fetch + cache, mai fetch diretti nei componenti
- Zustand solo per stato globale UI (filtri, utente auth) — non per dati server
- NativeWind per stili (classi Tailwind) — evita StyleSheet dove possibile
- Expo Router per tutta la navigazione — nessun React Navigation manuale
- Edge Functions in TypeScript (Deno runtime)
- Testa ogni hook con React Native Testing Library prima di usarlo nei componenti
- Prima di ogni Edge Function: verifica che la tabella Supabase esista già con la migration corrispondente
