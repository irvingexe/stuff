```mermaid
---
title: Arquitectura de calificación de leads, atención al cliente y captura de datos de venta (V3 Full Auto)
---
graph TD
    %% Entradas y Persistencia
    Lead((Lead)) -->|Mensajes Múltiples| Webhook[Webhook en n8n]
    
    subgraph Capa_Persistencia [Persistencia y Espera]
    Webhook --> SaveDB[(Base de Datos <br/>Supabase/Postgres)]
    SaveDB --> PreConfirm{¿Etapa previa a <br/>confirmación?}
    PreConfirm -- "No" --> End1([Fin del Flujo])
    PreConfirm -- "Sí" --> Debounce{¿Pasaron 45s <br/>sin mensajes nuevos?}
    Debounce -- "No" --> Wait[Esperar...]
    end

    %% Capa 0: Aduana y Contexto
    Debounce -- "Sí" --> GetHistory[Recuperar Mensajes <br/>No Procesados de DB]
    GetHistory --> Triage{Triage: Filtro Inicial <br/>Regex / n8n}
    
    Triage -- "Simple / Cierre" --> Ignore[Respuesta Estática / <br/>Marcar en DB]
    Triage -- "Complejo" --> GetContext[Extraer Estado y <br/>Resumen de Kommo]

    %% Capa 1: RAG Dinámico
    GetContext --> DetectIntent{¿Menciona <br/>Producto/Duda?}
    DetectIntent -- "Sí" --> FetchDoc[Fetch Ficha Técnica <br/>específica de DB]
    DetectIntent -- "No" --> OpenAICall
    FetchDoc --> OpenAICall[OpenAI API <br/>gpt-4o-mini]
    
    %% Capa 2: Salida y Acciones Paralelas
    OpenAICall --> |Prompt + Resumen + Doc + Historial| JSONResponse[Salida JSON <br/>Datos + Sentimiento + Mensaje]
    
    JSONResponse --> MarkDone[Marcar Mensajes como <br/>Procesados en DB]
    JSONResponse --> UpdateFields[Actualizar IA_Contexto_Resumen <br/>y Campos en Kommo]
    
    %% Capa 3: Lógica de Movimiento de Etapa
    UpdateFields --> EvalData{¿Estado de Datos <br/>para Pedido?}
    EvalData -- "Incompletos" --> MoveDatos[Mover a Etapa: <br/>'Datos']
    EvalData -- "Completos" --> MoveConfirm[Mover a Etapa: <br/>'Pendiente de Confirmación']
    EvalData -- "Sin cambios" --> SentimentCheck{¿Sentimiento <br/>Negativo?}

    MoveDatos --> SentimentCheck
    MoveConfirm --> SentimentCheck
    
    SentimentCheck -- "Sí" --> AlertAgent[Alerta a Humano <br/>Slack/Telegram]
    SentimentCheck -- "No" --> CheckHot{¿Etapa previa <br/>a caliente?}

    %% Capa 4: Ramificación Final
    CheckHot -- "No" --> End2([Fin del Flujo])
    CheckHot -- "Sí" --> SendMsg[Enviar Respuesta al Lead]
    
    %% Capa 5: Reporte Nocturno
    Cron((11:59 PM)) --> GatherLogs[Recopilar Historial <br/>del Día desde DB]
    GatherLogs --> BatchAPI[OpenAI Batch API <br/> 50% OFF]
    BatchAPI --> Report[Generar Reporte de <br/>Conversión y Métricas]

    %% ESTILOS
    style Webhook fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style SaveDB fill:#ffd700,stroke:#333,stroke-width:2px,color:#000
    style OpenAICall fill:#bbf,stroke:#333,stroke-width:2px,color:#000
    style JSONResponse fill:#fff,stroke:#333,stroke-dasharray: 5 5,color:#000
    style SendMsg fill:#00ff00,stroke:#333,stroke-width:2px,color:#000
    style End1 fill:#ffcccc,stroke:#333,color:#000
    style End2 fill:#ffcccc,stroke:#333,color:#000
    style AlertAgent fill:#ff4d4d,stroke:#333,color:#fff,stroke-width:2px
    style MoveDatos fill:#a1c4fd,stroke:#333,color:#000
    style MoveConfirm fill:#c2e9fb,stroke:#333,color:#000
```
