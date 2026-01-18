# Decisiones pendientes por orden de importancia

Puntos que bloquean o afectan el inicio del desarrollo.

---

## 1. Schema de entidades (CRÍTICO)

Afecta: modelo datos, API, app, IA. Bloquea inicio desarrollo.

### Alternativas

**A) Schema fijo predefinido** (más simple)
- Tipos hardcoded: `list`, `note`, `link`
- Cada tipo tiene campos fijos
- Fácil de implementar, UI predecible
- Limitado si usuario quiere tipos custom

```sql
-- Tabla por tipo
CREATE TABLE lists (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  title TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE list_items (
  id UUID PRIMARY KEY,
  list_id UUID REFERENCES lists(id) ON DELETE CASCADE,
  text TEXT NOT NULL
);

CREATE TABLE notes (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  title TEXT,
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE links (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  url TEXT NOT NULL,
  title TEXT,
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**B) Schema semi-flexible** (equilibrado)
- Tipos base predefinidos + campos JSONB para metadata custom
- Ej: `link` tiene `url`, `title` fijos + `tags`, `custom_fields` flexibles
- Balance entre estructura y flexibilidad

```sql
-- Tabla unificada con tipo + campos específicos + metadata flexible
CREATE TABLE entities (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  type TEXT NOT NULL CHECK (type IN ('list', 'note', 'link')),
  title TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),

  -- Campos específicos por tipo (nullable)
  content TEXT,           -- para notes
  url TEXT,               -- para links
  items JSONB,            -- para lists

  -- Campos flexibles para cualquier tipo
  tags TEXT[] DEFAULT '{}',
  metadata JSONB DEFAULT '{}'  -- custom fields usuario
);

-- Ejemplo metadata: {"priority": "alta", "project": "trabajo"}
```

**C) Schema totalmente dinámico** (más complejo)
- Usuario define sus propios tipos de entidad
- Campos definidos por usuario
- Máxima flexibilidad, más complejidad en UI y validación
- Riesgo: over-engineering para MVP

```sql
-- Tipos definidos por usuario
CREATE TABLE entity_types (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  name TEXT NOT NULL,           -- "receta", "contacto", "proyecto"
  schema JSONB NOT NULL         -- define campos y validación
);

-- Ejemplo schema: {
--   "fields": [
--     {"name": "ingredientes", "type": "array"},
--     {"name": "tiempo_prep", "type": "number"},
--     {"name": "dificultad", "type": "enum", "values": ["fácil","media","difícil"]}
--   ]
-- }

-- Entidades genéricas
CREATE TABLE entities (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  type_id UUID REFERENCES entity_types(id),
  data JSONB NOT NULL,          -- datos según schema del tipo
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 2. Modelo de embeddings (ALTO)

Afecta: búsqueda semántica, costes operativos, latencia.

### Alternativas

**A) OpenAI text-embedding-3-small** (más simple)
- API lista, sin infraestructura
- ~$0.02/1M tokens
- Dependencia externa
- Fácil de implementar

**B) Cohere embed-v3** (equilibrado)
- Mejor para multilingüe (español)
- API similar a OpenAI
- Tier gratuito generoso para MVP

**C) Self-hosted (sentence-transformers)** (más complejo)
- Sin costes por uso
- Requiere GPU o CPU potente
- Mayor control, más infraestructura
- Modelos: `paraphrase-multilingual-MiniLM-L12-v2`

---

## 3. API: REST vs GraphQL (MEDIO-ALTO)

Diagrama menciona ambos. Hay que decidir.

### Alternativas

**A) REST puro** (más simple)
- Endpoints claros: `/lists`, `/notes`, `/links`
- Tooling maduro, fácil debug
- Puede requerir múltiples requests

**B) GraphQL** (equilibrado)
- Un endpoint, queries flexibles
- Ideal si app necesita datos anidados
- Más setup inicial, curva aprendizaje

**C) REST + GraphQL parcial** (más complejo)
- REST para CRUD simple
- GraphQL para queries complejas/reportes
- Dos sistemas a mantener

---

## 4. Roadmap / fases MVP (MEDIO)

Sin fases claras, riesgo de scope creep.

### Alternativas

**A) MVP mínimo vertical** (más simple)
- Fase 1: Solo listas via Telegram + app read-only
- Fase 2: Añadir notas
- Fase 3: Añadir enlaces + búsqueda
- Validar cada fase con usuarios reales

**B) MVP horizontal básico** (equilibrado)
- Fase 1: Listas + notas + enlaces via Telegram (sin búsqueda semántica)
- Fase 2: App CRUD completa
- Fase 3: Búsqueda semántica + reglas usuario

**C) MVP completo** (más complejo)
- Todo el alcance definido de golpe
- Mayor riesgo, más tiempo hasta feedback
- Solo si hay certeza del producto

---

## 5. Proveedor LLM para clasificación (MEDIO)

Classifier, Structurer, Clarifier necesitan LLM.

### Alternativas

**A) OpenAI GPT-4o-mini** (más simple)
- Barato (~$0.15/1M input tokens)
- Muy capaz para clasificación
- API estable

**B) Anthropic Claude Haiku** (equilibrado)
- Similar coste
- Mejor en español según algunos benchmarks
- Buena opción si ya usas Claude

**C) Self-hosted (Llama 3, Mistral)** (más complejo)
- Sin costes por uso
- Requiere infraestructura GPU
- Más control, más mantenimiento

---

## 6. Vector DB para embeddings (MEDIO)

Si se usan embeddings, dónde almacenarlos.

### Alternativas

**A) pgvector en PostgreSQL** (más simple)
- Mismo DB, sin infra adicional
- Suficiente para <100k vectores
- Queries SQL normales

**B) Supabase (pgvector managed)** (equilibrado)
- PostgreSQL + pgvector hosted
- Auth incluido
- Tier gratuito generoso

**C) Pinecone / Qdrant dedicado** (más complejo)
- Optimizado para vectores
- Mejor performance a escala
- Otro servicio a gestionar

---

## Resumen recomendación rápida

| Decisión | Recomendación MVP |
|----------|-------------------|
| Schema entidades | B) Semi-flexible |
| Embeddings | A) OpenAI o B) Cohere |
| API | A) REST puro |
| Roadmap | A) MVP mínimo vertical |
| LLM | A) GPT-4o-mini |
| Vector DB | A) pgvector |

---

## Preguntas para decidir

1. ¿Cuánta flexibilidad necesita el usuario en tipos de entidad?
2. ¿Presupuesto mensual máximo para APIs externas en MVP?
3. ¿Preferencia por vendor (OpenAI vs Anthropic vs open source)?
4. ¿Primera funcionalidad a validar con usuarios: listas, notas o enlaces?
