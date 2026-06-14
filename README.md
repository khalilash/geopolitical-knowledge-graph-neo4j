# Geopolitical Knowledge Graph: Analysis with Neo4j, Wikidata & DBpedia

## Deskripsi Proyek

Proyek ini membangun knowledge graph geopolitik dengan mengintegrasikan data dari Wikidata dan DBpedia (kota, negara, benua, bahasa, mata uang) ke dalam Neo4j. Graph dianalisis menggunakan algoritma Graph Data Science (PageRank, Jaccard Similarity, Louvain Community Detection), serta diperkaya dengan komponen AI berbasis LangChain: Text-to-Cypher, LLM Graph Builder, dan GraphRAG.

## Arsitektur

```mermaid
flowchart TD
    A[Wikidata + DBpedia via SPARQL] --> B[CSV - Pandas Merge]
    B --> C[Neo4j Graph]
    C --> D[Graph Analytics<br/>PageRank, Jaccard, Louvain]
    C --> E[GraphCypherQAChain<br/>Text-to-Cypher and GraphRAG]
    C --> F[LLMGraphTransformer<br/>Unstructured text to Neo4j]
    F --> E
    D --> E
```

## Komponen

### 1. Data & Graph Schema
- **Node labels**: City, Country, Continent, Language, Currency
- **Relationships**: `BELONGS_TO`, `PART_OF`, `SPEAKS`, `USES_CURRENCY`
- Dataset: 250+ baris, mencakup 5 entitas, hasil integrasi Wikidata + DBpedia via SPARQL

### 2. Graph Analytics (Neo4j GDS)
- **PageRank**: mengukur dominasi struktural bahasa dalam jaringan
- **Jaccard Similarity**: mengukur kemiripan profil geopolitik antar negara
- **Louvain Community Detection**: mendeteksi blok/komunitas regional secara otomatis

### 3. LLM Text-to-Cypher & GraphRAG
Menggunakan `GraphCypherQAChain` dari LangChain. Chain ini secara otomatis:
1. Membaca schema graph via `graph.refresh_schema()`
2. Menerjemahkan pertanyaan bahasa natural menjadi query Cypher
3. Mengeksekusi query ke Neo4j
4. Menghasilkan jawaban natural language berdasarkan hasil query

### 4. LLM Graph Builder
Menggunakan `LLMGraphTransformer` untuk mengekstrak entitas dan relasi dari teks tidak terstruktur (contoh: artikel berita geopolitik). Ekstraksi dibatasi menggunakan `allowed_nodes` dan `allowed_relationships` agar hasil tetap konsisten dengan schema graph, kemudian ditambahkan ke Neo4j via `graph.add_graph_documents()`.

## Instalasi & Konfigurasi

### Step 1: Setup Database Neo4j

Jalankan Cypher berikut di Neo4j Browser (Sandbox/Aura/Desktop) untuk membuat schema constraint dan mengimpor dataset:

```cypher
CREATE CONSTRAINT FOR (c:City) REQUIRE c.name IS UNIQUE;
CREATE CONSTRAINT FOR (co:Country) REQUIRE co.name IS UNIQUE;
CREATE CONSTRAINT FOR (con:Continent) REQUIRE con.name IS UNIQUE;
CREATE CONSTRAINT FOR (l:Language) REQUIRE l.name IS UNIQUE;
CREATE CONSTRAINT FOR (cur:Currency) REQUIRE cur.name IS UNIQUE;

LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/khalilash/Dataset-Geopolitik-Graf/main/Geopolitik%20Dataset%20-%20Graf.csv' AS row

MERGE (city:City {name: row.cityName})
SET city.population = toInteger(row.population),
    city.coords = row.coords

MERGE (country:Country {name: row.countryName})
MERGE (continent:Continent {name: row.continent})
MERGE (lang:Language {name: row.officialLanguage})
MERGE (currency:Currency {name: row.currency})

MERGE (city)-[:BELONGS_TO]->(country)
MERGE (country)-[:PART_OF]->(continent)
MERGE (country)-[:SPEAKS]->(lang)
MERGE (country)-[:USES_CURRENCY]->(currency)
```

### Step 2: Install Python Dependencies
```bash
pip install langchain langchain-openai langchain-neo4j langchain-experimental
```

### Step 3: Environment Setup
```python
import os

os.environ["NEO4J_URI"] = "bolt+s://<your-instance>.neo4jsandbox.com:443"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "<your-password>"
os.environ["OPENROUTER_API_KEY"] = "<your-api-key>"
```

> Catatan: Jangan commit API key/password ke repository. Gunakan environment variable atau file `.env`.

### Cara Menjalankan
1. Jalankan Step 1 di Neo4j Browser (sekali saja, saat setup database)
2. Buka `notebook.ipynb` di Google Colab
3. Jalankan cell instalasi library (Step 2)
4. Isi kredensial Neo4j dan OpenRouter pada cell konfigurasi (Step 3)
5. Jalankan seluruh cell secara berurutan (Run All)
## Logika Cypher & Pipeline AI

### GraphCypherQAChain (Text-to-Cypher + GraphRAG)
Chain ini diinisialisasi dengan `graph` (koneksi Neo4j) dan `llm` (model via OpenRouter). Saat menerima pertanyaan, chain secara otomatis menghasilkan Cypher query (`cypher_llm`), mengeksekusinya, lalu memformulasikan jawaban natural language (`qa_llm`) berdasarkan hasil query — tanpa perlu menulis prompt schema secara manual karena schema dibaca otomatis dari Neo4j.

### LLMGraphTransformer (Graph Builder)
Teks tidak terstruktur diubah menjadi `Document`, kemudian diproses oleh `LLMGraphTransformer` yang dikonfigurasi dengan `allowed_nodes` (Country, City, Organization, Currency, Region) dan `allowed_relationships` (MEMBER_OF, CAPITAL_OF, LOCATED_IN, TRADES_WITH, USES_CURRENCY). Hasil ekstraksi berupa graph documents yang langsung dapat ditambahkan ke Neo4j.

## Penggunaan AI dalam Pengembangan

Proyek ini dikembangkan dengan bantuan AI assistant (Claude, Anthropic) untuk membantu perancangan arsitektur pipeline, debugging kode, dan penulisan dokumentasi.

### Model yang digunakan
- **AI Assistant (development)**: Claude (Anthropic)
- **LLM dalam pipeline**: `qwen/qwen-2.5-72b-instruct` via OpenRouter — digunakan oleh `GraphCypherQAChain` dan `LLMGraphTransformer`

### Prompt utama yang digunakan
- Merancang arsitektur pipeline Tier 4 (Graph Analytics, Text-to-Cypher, LLM Graph Builder, GraphRAG)
- Debugging error koneksi Neo4j (URI scheme) dan rate limit OpenRouter
- Menyusun prompt internal untuk Text-to-Cypher (schema graph → Cypher) dan GraphRAG (hasil query → jawaban natural language)

### Modifikasi Manual
- Penyesuaian kredensial koneksi (Neo4j Sandbox URI, OpenRouter API key) ke instance dan dataset milik kelompok ini
- Penggantian model LLM akibat unavailable/rate-limited
- Pengujian query dengan berbagai pertanyaan untuk validasi hasil
- Penyesuaian `allowed_nodes` dan `allowed_relationships` agar sesuai schema graph

## Dataset & Sumber

- Wikidata SPARQL endpoint: https://query.wikidata.org/
- DBpedia SPARQL endpoint: https://dbpedia.org/sparql
- Dataset hasil integrasi: `Geopolitik Dataset - Graf UAS.csv`

## Tim
- Khalila Shafarayhani Atletiko - 5026231167
- Alisha Rafimalia - 5026231202
