```mermaid
graph TD
    Lead((Lead)) -->|Mensajes Múltiples| Webhook[Webhook en n8n]
    
    subgraph Capa_Persistencia [Persistencia y Espera]
    Webhook --> SaveDB[(Base de Datos <br/>Supabase/Postgres)]
    SaveDB --> Debounce{¿Pasaron 45s <br/>sin mensajes nuevos?}
    end

    Debounce -- "Sí" --> GetHistory[Recuperar Mensajes <br/>No Procesados de DB]
    Debounce -- "No" --> Wait[Esperar...]

    GetHistory --> CheckState{¿Etapa del Lead?}
    CheckState -- "Frío / Caliente" --> OpenAICall[OpenAI API <br/>gpt-4o-mini]
    
    OpenAICall --> JSONResponse[Salida JSON]
    
    JSONResponse --> SendMsg[Enviar Respuesta al Lead]
    JSONResponse --> MarkDone[Marcar Mensajes como <br/>Procesados en DB]
    JSONResponse --> UpdateKommo[Actualizar Campos <br/>en Kommo]

    %% Estilos
    style Webhook fill:#f9f,stroke:#333,stroke-width:2px,color:#000
    style SaveDB fill:#ffd700,stroke:#333,stroke-width:2px,color:#000
    style OpenAICall fill:#bbf,stroke:#333,stroke-width:2px,color:#000
```
