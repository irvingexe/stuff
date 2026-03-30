```mermaid
---
title: Arquitectura de calificación de leads, atención al cliente y captura de datos de venta
---
graph TD
    %% Entradas y Persistencia
    Lead((Lead)) -->|Mensajes Múltiples| Webhook[Webhook en n8n]
    
    subgraph Capa_Persistencia ["💰 Persistencia, filtrado y Espera antes de procesar"]
    Webhook --> SaveDB[(Base de Datos <br/>Supabase/Postgres)]
    SaveDB --> PreConfirm{¿Etapa previa a <br/>confirmación?}
    PreConfirm -- "No" --> End1([Fin del Flujo])
    PreConfirm -- "Sí" --> Debounce{¿Pasaron 45s <br/>sin mensajes nuevos?}
    Debounce -- "No" --> Wait[Esperar...]
    end

    %% Capa 0: Aduana y Contexto
    Debounce -- "Sí" --> GetHistory[Recuperar Mensajes <br/>No Procesados de DB]
    GetHistory --> Triage{"Triage: Filtro Inicial <br/>Regex / n8n <br/><b>💰 Evitar procesar mensajes que no aportan datos ('Ok', 'Gracias', etc.)"}
    Triage -- "Simple / Cierre" --> Template{¿Requiere respuesta?}
    Template -- "No" --> End3([Fin del Flujo])
    Template -- "Sí" --> Ignore[Respuesta Estática / <br/>Marcar en DB]
    Triage -- "Complejo" --> GetContext[Extraer Estado, Asesor <br/>y Resumen de Kommo]

    %% Capa 1: RAG Dinámico Condicional
    GetContext --> CheckAuto{¿En etapas de <br/>atención automática? <br/><b>💰 Responder solo si es necesario<b/>}
    
    CheckAuto -- "No" --> OpenAICall[OpenAI API <br/>gpt-4o-mini]
    
    CheckAuto -- "Sí" --> DetectIntent{¿Menciona <br/>Producto/Duda?}
    
    DetectIntent -- "Sí" --> FetchDoc[💰 Incluir ficha técnica de producto/información de la empresa en prompt solo si es necesario]
    DetectIntent -- "No" --> OpenAICall
    FetchDoc --> OpenAICall
    
    %% Capa 2: Salida y Acciones Paralelas
    OpenAICall --> |Prompt + Resumen + Doc + Historial| JSONResponse[Salida JSON: Datos + Sentimiento <br/>+ Temperatura + Mensaje]
    
    JSONResponse --> MarkDone[Marcar Mensajes como <br/>Procesados en DB]
    JSONResponse --> UpdateFields[Actualizar IA_Contexto_Resumen <br/>y Campos en Kommo]
    
    %% Capa 3: Lógica de Temperatura y Etapas
    UpdateFields --> EvalTemp{¿Temperatura <br/>del Lead?}
    EvalTemp -- "Frío" --> MoveFrio[Mover a Etapa: <br/>'Frío']
    EvalTemp -- "Caliente" --> MoveCaliente[Mover a Etapa: <br/>'Caliente']

    MoveFrio --> EvalData{¿Estado de Datos <br/>para Pedido?}
    MoveCaliente --> EvalData
    
    EvalData -- "Incompletos" --> MoveDatos[Mover a Etapa: <br/>'Datos']
    EvalData -- "Completos" --> MoveConfirm[Mover a Etapa: <br/>'Pendiente de Confirmación']
    EvalData -- "Sin cambios" --> LogMetric[Registrar Métrica en <br/>DB para Reporte Diario]

    MoveDatos --> LogMetric
    MoveConfirm --> LogMetric
    
    %% Capa 4: Alertas y Respuesta
    LogMetric --> SentimentCheck{¿Sentimiento <br/>Negativo?}
    SentimentCheck -- "Sí" --> AlertAgent[Alerta a Humano <br/>Tarea de Kommo/Whatsapp]
    SentimentCheck -- "No" --> CheckHot{¿Se generó mensaje <br/>de respuesta?}

    CheckHot -- "No" --> End2([Fin del Flujo])
    CheckHot -- "Sí" --> SendMsg[Enviar Respuesta al Lead]
    
    %% Capa 5: Reporte Nocturno (Auditoría de Calidad)
    Cron((11:59 PM)) --> GatherLogs[Recopilar Logs, Timestamps <br/>y Mensajes del Día]
    GatherLogs --> BatchAPI[OpenAI Batch API: <br/>Análisis de Desempeño Asesor <br/><b>💰 Ahorro del 50%<b/>]
    BatchAPI --> Report[Generar REPORTE DIARIO IA]

    %% ESTILOS
    style BatchAPI fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style Triage fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style Capa_Persistencia fill:#f9f,stroke:#333,stroke-width:2px,color:#000,opacity:0.4
    style FetchDoc fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style UpdateFields fill:#a1c4fd,stroke:#333,color:#000
    style Ignore fill:#00ff00,stroke:#333,stroke-width:2px,color:#fff,opacity:0.5

    %%style Webhook fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style SaveDB fill:#ffd700,stroke:#333,stroke-width:2px,color:#000
    style OpenAICall fill:#fff,stroke:#333,stroke-width:2px,color:#000,opacity:0.8
    style JSONResponse fill:#fff,stroke:#333,stroke-dasharray: 5 5,color:#000
    style SendMsg fill:#00ff00,stroke:#333,stroke-width:2px,color:#fff,opacity:0.5
    style End1 fill:#ffcccc,stroke:#333,color:#000
    style End2 fill:#ffcccc,stroke:#333,color:#000
    style End3 fill:#ffcccc,stroke:#333,color:#000
    style AlertAgent fill:#ff4d4d,stroke:#333,color:#fff,stroke-width:2px
    style MoveDatos fill:#a1c4fd,stroke:#333,color:#000
    style MoveConfirm fill:#a1c4fd,stroke:#333,color:#000
    style MoveFrio fill:#a1c4fd,stroke:#333,color:#000
    style MoveCaliente fill:#a1c4fd,stroke:#333,color:#000
    style LogMetric fill:#fff,stroke:#333,stroke-dasharray: 2 2,color:#000
    style CheckAuto fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    ```
