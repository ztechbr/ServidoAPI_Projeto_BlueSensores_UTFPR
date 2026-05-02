# Servidor API — Projeto BlueSensores (UTFPR)

API em Python com **Flask**, documentação **Swagger** para o REST e um **Web Service SOAP 1.1** no mesmo path **`/soap`**, pensada para **testes de integração** com o **aplicativo Android de sensores** e para clientes legados. A API recebe leituras em JSON via **`POST /leituras`**, grava na tabela PostgreSQL `leituras`, e permite **consultas filtradas** por:

- **`GET /leituras`** (REST JSON; opcionalmente protegido por **`API_TOKEN`**),
- **`GET /soap?format=json`** ou **`format=xml`** (mesmos filtros na query string; **sem** token; útil no navegador),
- **`POST /soap`** com envelope **SOAP 1.1** ou **`GET /soap?wsdl`** para o contrato (também **sem** `API_TOKEN`).

As regras de filtro são as mesmas em todos os casos.

**Repositório no GitHub:** use o nome **`Servidor_API_Projeto_BlueSensores_UTFPR`**. Para renomear um repositório já criado: *Settings → General → Repository name*.

> **Escopo:** ambiente de desenvolvimento e testes — não use esta configuração (debug, servidor embutido) em produção sem endurecimento adequado (HTTPS, autenticação, processo WSGI, etc.).

## Pré-requisitos

- Python 3.10+ (recomendado)
- PostgreSQL com a tabela criada conforme `scripts_bd/create_table.sql`
- Se o banco **já existia** antes da inclusão dos campos elétricos, aplique também `scripts_bd/alter_leituras_add_eletricos.sql` uma vez
- Rede acessível entre o celular/emulador e a máquina que roda a API (mesma Wi‑Fi ou túnel)

## Configuração do banco (`.env`)

Na raiz do projeto, copie o modelo e ajuste:

```bash
cp .env.example .env
```

Você pode usar **uma** das formas:

1. **URL completa:** `DATABASE_URL=postgresql://usuario:senha@host:5432/nome_do_banco`
2. **Variáveis separadas:** `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`

Opcional:

| Variável | Descrição |
|----------|-----------|
| `PORT` | Porta HTTP da API (padrão **8001**). |
| `API_TOKEN` | Se definido e não vazio, **`GET`/`POST /leituras`** exigem `Authorization: Bearer <token>` ou cabeçalho `X-API-Key`. **Não** se aplica a **`/soap`** nem a **`/health`** / **`/apidocs`**. |
| `SOAP_PUBLIC_URL` | URL pública do endpoint SOAP (ex.: `https://seu-dominio/soap`). Fixa `<soap:address location="..."/>` no WSDL; se **`SOAP_NAMESPACE`** estiver vazio, o *target namespace* do WSDL vira `{scheme}://{host}/leituras`. |
| `SOAP_NAMESPACE` | *Target namespace* explícito no WSDL (opcional; sobrescreve a derivação a partir de `SOAP_PUBLIC_URL`). |

Veja comentários no arquivo **`.env.example`**.

**Dependências:** o SOAP usa **Spyne** (instalação via Git no Python 3.12+) e **`lxml`**; o `Dockerfile` instala **Git** só durante o `pip install`.

## Instalação e execução

```bash
cd Servidor_API_Projeto_BlueSensores_UTFPR
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

Por padrão o servidor sobe em **`http://0.0.0.0:8001`** (acessível na rede local pelo IP da máquina).

## Endpoints

| Método | Caminho | Descrição |
|--------|---------|-----------|
| `GET` | `/health` | Verifica se o serviço está no ar (`{"status":"ok"}`). |
| `GET` | `/leituras` | Lista leituras com filtros (obrigatório pelo menos um filtro). |
| `POST` | `/leituras` | Insere uma leitura (JSON). |
| `GET` | `/soap` | Depende dos parâmetros: **`?wsdl`** → WSDL; **`?format=json`** ou **`format=xml`** + filtros → mesma consulta que `GET /leituras`; **sem filtros** → JSON de ajuda. **Não** usa `API_TOKEN`. |
| `POST` | `/soap` | Consulta via **SOAP 1.1** (corpo XML; operação `listarLeituras`). **Não** usa `API_TOKEN`. |

### Como escolher: REST (Swagger) ou Web Service (SOAP)

| Objetivo | Caminho recomendado |
|----------|----------------------|
| Explorar e testar a API no navegador, ver parâmetros e exemplos | **Swagger** em `/apidocs` (use **Authorize** se `API_TOKEN` estiver configurado) |
| Integrar apps modernos, mobile ou scripts com JSON | **REST** (`GET`/`POST` em `/leituras`) com token se exigido |
| Ver dados no navegador sem Postman, em JSON ou XML | **`GET /soap?format=json` ou `format=xml`** com os mesmos query params do `GET /leituras` |
| Integrar sistemas legados ou ferramentas SOAP | **WSDL** em `/soap?wsdl` e **`POST /soap`** com envelope XML |

A **gravação** de novas leituras está disponível **apenas por REST** (`POST /leituras`). A **leitura com filtros** pode ser feita por **REST** ou **SOAP**, com o mesmo significado de filtros.

---

### Documentação REST — Swagger UI

1. Com o servidor em execução, abra no navegador: **`http://<host>:<porta>/apidocs`**  
   (ex.: `http://127.0.0.1:8001/apidocs` ou `http://localhost:8001/apidocs` com Docker).
2. A interface **Swagger UI** lista os endpoints (`/health`, `GET /leituras`, `POST /leituras`), os parâmetros (query, body) e os códigos de resposta descritos no projeto. **`/soap`** não aparece no Swagger (é SOAP/GET alternativo; use as URLs descritas abaixo).
3. Se o servidor tiver **`API_TOKEN`** definido, clique em **Authorize** e informe **`Bearer <seu_token>`** (o mesmo valor da variável de ambiente) antes de testar **`GET`/`POST /leituras`**.
4. Para **experimentar** uma rota: expanda a operação → **Try it out** → preencha parâmetros ou o JSON do corpo → **Execute**.
5. A própria UI mostra o **curl** gerado e o corpo da resposta, o que ajuda a repetir a chamada em Postman, no app ou em scripts.

Assim você valida contratos e URLs sem escrever código à mão.

### Autenticação REST (`API_TOKEN`)

Quando **`API_TOKEN`** está definido no `.env` (valor não vazio), inclua em **`GET /leituras`** e **`POST /leituras`** um dos seguintes:

- Cabeçalho **`Authorization: Bearer <API_TOKEN>`**, ou  
- Cabeçalho **`X-API-Key: <API_TOKEN>`**

Se **`API_TOKEN`** estiver vazio ou ausente, essas rotas permanecem abertas (adequado para desenvolvimento local). **`/soap`**, **`/health`** e **`/apidocs`** não usam esse token.

---

### Web Service e `/soap` — consulta de leituras

No mesmo path **`/soap`** a API oferece:

1. **WSDL** — `GET /soap?wsdl` (contrato XML para importar em clientes SOAP).  
2. **GET com JSON ou XML** — query string igual ao `GET /leituras`, mais **`format=json`** (padrão) ou **`format=xml`**. Ex.: `/soap?format=json&codplantacao=PLANTDEMO`. Sem filtros obrigatórios, responde um JSON de ajuda.  
3. **POST SOAP 1.1** — envelope XML, operação **`listarLeituras`**.

Em todos os casos valem os **mesmos filtros** (pelo menos um entre `codplantacao`, `dataleit_inicio`, `dataleit_fim`; `limit` e `offset` opcionais). Nenhuma dessas rotas usa **`API_TOKEN`**.

**Namespace versus URL do serviço:** o *target namespace* em `<xs:schema targetNamespace="..."/>` identifica os tipos no XML; a chamada HTTP usa **`<soap:address location="..."/>`**. Defina **`SOAP_PUBLIC_URL`** no `.env` com a URL pública do serviço (ex.: `https://api.exemplo.com/soap`). Se **`SOAP_NAMESPACE`** não estiver definido e **`SOAP_PUBLIC_URL`** estiver, o namespace é derivado como **`{scheme}://{host}/leituras`**. Sem nenhum dos dois, o WSDL mantém o identificador padrão do projeto (`http://utfpr.edu.br/bluesensores/leituras`). Para forçar outro namespace, use **`SOAP_NAMESPACE`**.

| Item | Valor |
|------|--------|
| **WSDL** | `https://<host>/soap?wsdl` (ou `/soap/?wsdl`) |
| **GET JSON/XML** | `GET /soap?format=json&…` ou `format=xml&…` (mesmos parâmetros que `GET /leituras`) |
| **POST SOAP** | `POST /soap` — corpo: envelope SOAP 1.1 |
| **Operação (POST)** | `listarLeituras` |
| **Namespace XML (`tns`)** | Atributo `targetNamespace` do WSDL (veja `SOAP_NAMESPACE` / derivação acima) |
| **Cabeçalhos (POST)** | `Content-Type: text/xml; charset=utf-8` e, em geral, `SOAPAction: listarLeituras` |

**Teste no navegador:** abra por exemplo `http://127.0.0.1:8001/soap?format=json&codplantacao=PLANTDEMO`. Para só ver o WSDL: `http://127.0.0.1:8001/soap?wsdl`.

No **POST SOAP**, o elemento **`filtro`** aceita os mesmos campos que a query do REST: `codplantacao`, `dataleit_inicio`, `dataleit_fim`, `limit`, `offset`. Sem filtro válido, o POST retorna **SOAP Fault** (equivalente ao `GET /leituras` sem filtros).

Exemplo de envelope (ajuste host, porta e valores):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap11env:Envelope xmlns:soap11env="http://schemas.xmlsoap.org/soap/envelope/">
  <soap11env:Body>
    <tns:listarLeituras xmlns:tns="http://utfpr.edu.br/bluesensores/leituras">
      <tns:filtro>
        <tns:codplantacao>PLANTDEMO</tns:codplantacao>
        <tns:dataleit_inicio>2026-05-01</tns:dataleit_inicio>
        <tns:dataleit_fim>2026-05-31</tns:dataleit_fim>
        <tns:limit>100</tns:limit>
        <tns:offset>0</tns:offset>
      </tns:filtro>
    </tns:listarLeituras>
  </soap11env:Body>
</soap11env:Envelope>
```

Use no `xmlns:tns` o mesmo valor de **`targetNamespace`** que aparecer no WSDL obtido em **`GET /soap?wsdl`** (depende de `SOAP_NAMESPACE` / `SOAP_PUBLIC_URL`).

**Sugestão de teste rápido com curl** (envia o XML acima salvo em `request.xml`):

```bash
curl -s -X POST "http://127.0.0.1:8001/soap" \
  -H "Content-Type: text/xml; charset=utf-8" \
  -H "SOAPAction: listarLeituras" \
  --data-binary @request.xml
```

Em **.NET**, **Java** (JAX-WS, CXF) ou **SoapUI** / **Postman**, use **Importar WSDL** a partir da URL `/soap/?wsdl` e gere ou configure o cliente apontando para o mesmo host e porta da API.

> **Nota:** a instalação do projeto pode exigir **Git** para baixar a dependência SOAP (Spyne) no Python 3.12+; o `Dockerfile` já instala o pacote `git` só durante o build da imagem.

---

### GET `/leituras` — filtros (query string)

Informe **pelo menos um** dos filtros abaixo:

- `codplantacao` — código da plantação
- `dataleit_inicio` — data inicial inclusiva (`YYYY-MM-DD`)
- `dataleit_fim` — data final inclusiva (`YYYY-MM-DD`)

Opcionais: `limit` (1–500, padrão 100), `offset` (paginação).

Exemplo:

```text
GET http://192.168.1.10:8001/leituras?codplantacao=PLANTDEMO&dataleit_inicio=2026-05-01&dataleit_fim=2026-05-31
```

Com **`API_TOKEN`** configurado, inclua por exemplo `-H "Authorization: Bearer SEU_TOKEN"` no `curl` ou no cliente HTTP.

### POST `/leituras` — corpo JSON

Campos **obrigatórios:** `codplantacao`, `codleitura`, `lat`, `lon`, `dataleit`, `horaleit`.

- `dataleit`: string `YYYY-MM-DD`
- `horaleit`: string `HH:MM` ou `HH:MM:SS`

Demais campos numéricos são opcionais; se omitidos, a API usa o valor sentinela **-9999** (alinhado aos defaults da tabela). Entre eles estão os sensores ambientais habituais e, opcionalmente, **comunicação / grandezas elétricas:** `scomunicacao`, `stensao`, `scorrente`, `spotencia`. `status_blockchain` pode ser `PENDENTE`, `ENVIADO` ou `CONFIRMADO` (padrão `PENDENTE`).

Exemplo com `curl` (adicione `-H "Authorization: Bearer SEU_TOKEN"` se o servidor usar `API_TOKEN`):

```bash
curl -X POST "http://127.0.0.1:8001/leituras" \
  -H "Content-Type: application/json" \
  -d '{
    "codplantacao": "PLANTDEMO",
    "codleitura": "LEIT001",
    "lat": -22.9068,
    "lon": -43.1729,
    "dataleit": "2026-05-01",
    "horaleit": "14:30:00",
    "temp_solo": 25.5,
    "temp_ar": 28.3,
    "umid_solo": 60.2,
    "umid_ar": 55.1,
    "luz": 800.0,
    "chuva": 0.0,
    "umid_folha": 10.5,
    "scomunicacao": 1.0,
    "stensao": 220.0,
    "scorrente": 0.5,
    "spotencia": 110.0,
    "status_blockchain": "PENDENTE"
  }'
```

Respostas comuns: **201** (criado, retorna `hash_pk`), **400** (validação), **409** (leitura duplicada pela chave gerada), **500** (erro de banco ou conexão).

---

## Android: enviar leitura com `POST` em Kotlin

No app de sensores, use a URL base apontando para o computador que executa o Flask:

- **Emulador Android:** para o `localhost` da máquina host, use **`http://10.0.2.2:<PORTA>`** (por exemplo `8001`).
- **Dispositivo físico:** use o **IP da máquina na LAN** (ex.: `http://192.168.1.10:8001`). O celular e o PC devem estar na mesma rede (ou use um túnel tipo ngrok).

No `AndroidManifest.xml`, declare permissão de internet:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

Se usar **HTTP** (não HTTPS) em testes, pode ser necessário permitir cleartext ou configurar *Network Security Config* para o domínio/IP de desenvolvimento (somente em debug).

### Exemplo com OkHttp

No **`/apidocs`** (Swagger), o corpo de **`POST /leituras`** já lista os campos opcionais numéricos, inclusive **`scomunicacao`**, **`stensao`**, **`scorrente`** e **`spotencia`** — o mesmo contrato usado abaixo.

Adicione no `build.gradle` do módulo (versões podem ser atualizadas):

```kotlin
dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

Exemplo de função (execute em `Dispatchers.IO` dentro de uma coroutine, ou use `enqueue` do OkHttp para não bloquear a UI):

```kotlin
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody
import org.json.JSONObject
import java.time.LocalDate
import java.time.LocalTime
import java.time.format.DateTimeFormatter

fun enviarLeitura(
    baseUrl: String, // ex: "http://10.0.2.2:8001" ou "http://192.168.1.10:8001"
    codPlantacao: String,
    codLeitura: String,
    lat: Double,
    lon: Double,
    tempSolo: Double?,
    tempAr: Double?,
    scomunicacao: Double? = null,
    stensao: Double? = null,
    scorrente: Double? = null,
    spotencia: Double? = null,
): Result<String> = runCatching {
    val hoje = LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE)
    val agora = LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss"))

    val json = JSONObject().apply {
        put("codplantacao", codPlantacao)
        put("codleitura", codLeitura)
        put("lat", lat)
        put("lon", lon)
        put("dataleit", hoje)
        put("horaleit", agora)
        tempSolo?.let { put("temp_solo", it) }
        tempAr?.let { put("temp_ar", it) }
        scomunicacao?.let { put("scomunicacao", it) }
        stensao?.let { put("stensao", it) }
        scorrente?.let { put("scorrente", it) }
        spotencia?.let { put("spotencia", it) }
        put("status_blockchain", "PENDENTE")
    }

    val client = OkHttpClient()
    val body = json.toString().toRequestBody("application/json; charset=utf-8".toMediaType())
    val request = Request.Builder()
        .url("$baseUrl/leituras")
        .post(body)
        .build()

    client.newCall(request).execute().use { response ->
        val texto = response.body?.string().orEmpty()
        if (!response.isSuccessful) {
            error("HTTP ${response.code}: $texto")
        }
        texto
    }
}
```

Pontos importantes:

1. **`Content-Type: application/json`** — o exemplo usa `application/json; charset=utf-8` no `RequestBody`, compatível com a API. Se a API exigir **`API_TOKEN`**, adicione `.header("Authorization", "Bearer $token")` (ou `X-API-Key`) ao `Request`.
2. **Datas e hora** — a API espera `dataleit` como `YYYY-MM-DD` e `horaleit` como `HH:MM` ou `HH:MM:SS`; o exemplo usa data/hora atuais do dispositivo.
3. **Thread** — `execute()` bloqueia a thread atual; em Activity chame de uma coroutine com `withContext(Dispatchers.IO) { enviarLeitura(...) }` ou use `enqueue` do OkHttp.
4. **Produção** — troque HTTP por HTTPS, valide certificados e adicione autenticação se a API for exposta na internet.

Com isso você integra o app Android de sensores aos testes desta API Flask de forma alinhada ao modelo da tabela `leituras`.

---

## Rodando com Docker (alternativa à linha de comando)

Além de instalar Python na máquina e executar `python app.py`, você pode subir **API + PostgreSQL** com **Docker Compose**. O repositório inclui um `Dockerfile` (imagem da API) e um `docker-compose.yml` que:

- sobe o PostgreSQL e, na **primeira** inicialização do volume, aplica `scripts_bd/create_table.sql` para criar a tabela `leituras`;
- constrói e inicia a API Flask na porta **8001** dentro da rede do Compose, apontando `DATABASE_URL` para o serviço `db`.

### Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) e [Docker Compose](https://docs.docker.com/compose/) (plugin `docker compose`).

### Comandos

Na raiz do projeto:

```bash
docker compose build
docker compose up -d
```

- Swagger: `http://localhost:8001/apidocs` (ajuste a porta se usar `API_PORT`).
- WSDL SOAP: `http://localhost:8001/soap?wsdl`.
- Exemplo GET leve no navegador: `http://localhost:8001/soap?format=json&codplantacao=PLANTDEMO`.
- O Postgres fica exposto no host na porta **5432** por padrão (`POSTGRES_PORT`).

### Variáveis opcionais

Você pode definir no ambiente ou num arquivo `.env` **na pasta do projeto** (usado pelo Compose para interpolação):

| Variável | Padrão | Uso |
|----------|--------|-----|
| `POSTGRES_USER` | `bluet` | usuário do banco |
| `POSTGRES_PASSWORD` | `bluet_secret` | senha |
| `POSTGRES_DB` | `bluet` | nome do banco |
| `POSTGRES_PORT` | `5432` | porta do Postgres no host |
| `API_PORT` | `8001` | porta da API no host |
| `API_TOKEN` | *(vazio)* | Repasse opcional; mesma regra do `.env` para REST `/leituras`. |
| `SOAP_PUBLIC_URL` | *(vazio)* | URL pública do SOAP/WSDL em produção (ex.: `https://dominio/soap`). |
| `SOAP_NAMESPACE` | *(vazio)* | *Target namespace* do WSDL; se vazio e `SOAP_PUBLIC_URL` definido, deriva `{scheme}://{host}/leituras`. |

A API dentro do contêiner usa `DATABASE_URL` gerado automaticamente a partir dos valores do Postgres e do hostname `db`.

### Observações

- Para **parar** e remover contêineres: `docker compose down`. Para apagar também o volume do Postgres (apaga os dados): `docker compose down -v`.
- Se o volume do banco **já existir** de uma execução anterior, o script de `initdb` **não roda de novo**. Nesse caso, garanta que a tabela exista (ou recrie o volume, ciente de que os dados serão perdidos). Se a tabela foi criada sem as colunas `scomunicacao`, `stensao`, `scorrente` e `spotencia`, execute `scripts_bd/alter_leituras_add_eletricos.sql` no Postgres (por exemplo com `psql` apontando para o serviço `db`).

Assim você pode testar o mesmo fluxo do app Android e os endpoints REST usando apenas Docker, sem precisar configurar Python e Postgres diretamente no sistema operacional.
