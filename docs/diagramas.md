# Diagramas do OctoFit Tracker

Este documento detalha como as partes do OctoFit Tracker se conectam e onde cada responsabilidade está organizada no repositório.

## Diagrama de arquitetura

```mermaid
flowchart LR
    pessoa[Pessoa usuária] --> browser[Navegador]
    browser -->|interface| vite["Frontend<br/>React 19 + Vite<br/>porta 5173"]
    vite --> router["App.tsx<br/>React Router"]
    router --> components["Activities · Teams<br/>Leaderboard · Workouts"]
    components -. "integração prevista" .-> client["src/api/client.ts<br/>getJson('/api/...')"]
    client -->|HTTP /api| proxy["Proxy do Vite<br/>vite.config.ts"]
    proxy -->|localhost:8000| express["API Express<br/>backend/src/app.ts"]
    express --> routes["Rotas /api<br/>backend/src/routes/api.ts"]
    routes --> services["Serviços<br/>backend/src/services/octofitService.ts"]
    services --> models["Modelos Mongoose<br/>User · Team · Activity · Workout"]
    models -->|MongoDB| database[("MongoDB<br/>porta 27017")]
    services -->|resultado ou erro| routes
    routes --> express
    express -->|resposta HTTP| browser
    server["backend/src/server.ts"] -->|conecta antes de iniciar| database
    server --> express
    health["GET /health"] --> express

    subgraph verificacao["Testes de contrato"]
        frontendTest["frontend/tests/app.contract.test.tsx"]
        apiTest["backend/tests/api.contract.test.ts"]
        dataTest["backend/tests/data.contract.test.ts"]
    end
    frontendTest -. verifica .-> components
    apiTest -. verifica .-> routes
    dataTest -. verifica .-> models
```

O navegador usa caminhos relativos `/api`; durante o desenvolvimento, o Vite encaminha essas chamadas da porta `5173` para o Express na porta `8000`. O backend usa Mongoose para acessar o MongoDB na porta `27017`. `VITE_API_BASE_URL` pode substituir a URL-base do cliente.

## Fluxograma da estrutura do projeto

```mermaid
flowchart TB
    root["octofit-tracker/"]

    root --> frontend["frontend/ — interface"]
    frontend --> fsrc["src/"]
    fsrc --> main["main.tsx<br/>inicializa React e BrowserRouter"]
    fsrc --> app["App.tsx<br/>rotas da interface"]
    fsrc --> api["api/client.ts<br/>cliente HTTP compartilhado"]
    fsrc --> components["components/"]
    components --> activities["Activities.tsx"]
    components --> teams["Teams.tsx"]
    components --> leaderboard["Leaderboard.tsx"]
    components --> workouts["Workouts.tsx"]
    frontend --> vite["vite.config.ts<br/>porta 5173 e proxy /api"]
    frontend --> ftests["tests/app.contract.test.tsx"]

    root --> backend["backend/ — API e dados"]
    backend --> bsrc["src/"]
    bsrc --> server["server.ts<br/>conecta banco e inicia API"]
    bsrc --> serverApp["app.ts<br/>middleware, /health e /api"]
    bsrc --> config["config/<br/>database.ts · urls.ts"]
    bsrc --> routes["routes/api.ts<br/>endpoints HTTP"]
    bsrc --> services["services/octofitService.ts<br/>regras de aplicação"]
    bsrc --> models["models/<br/>schemas e modelos Mongoose"]
    models --> user["User.ts"]
    models --> team["Team.ts"]
    models --> activity["Activity.ts"]
    models --> workout["Workout.ts"]
    bsrc --> seed["scripts/seed.ts<br/>dados de demonstração"]
    backend --> btests["tests/<br/>api.contract.test.ts · data.contract.test.ts"]

    services -->|consulta e grava| models
    routes -->|invoca| services
    api -->|chama /api via proxy| routes
```

> **Estado atual:** o projeto contém a estrutura e os contratos dos recursos, mas partes ainda são scaffolding. Os componentes de capacidade exibem “Implementation pending” e ainda não chamam o cliente de API. No backend, criação de atividades e equipes responde `501`, e o leaderboard ainda retorna uma lista vazia.
