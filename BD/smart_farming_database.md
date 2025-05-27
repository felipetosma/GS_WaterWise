# Modelo de Banco de Dados - Smart Farming IoT
## Sistema de Irrigação Inteligente por Zonas

### 📊 **Modelagem Relacional Otimizada (Oracle)**

```sql
-- Tabela de Usuários/Fazendeiros
CREATE TABLE GS_WW_USUARIO (
    id              NUMBER PRIMARY KEY,
    nome            VARCHAR2(100) NOT NULL,
    email           VARCHAR2(100) UNIQUE NOT NULL,
    senha_hash      VARCHAR2(255) NOT NULL,
    telefone        VARCHAR2(20),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status          VARCHAR2(20) DEFAULT 'ATIVO',
    last_login      TIMESTAMP
);

-- Tabela de Propriedades Rurais
CREATE TABLE GS_WW_PROPRIEDADE (
    id              NUMBER PRIMARY KEY,
    nome            VARCHAR2(100) NOT NULL,
    endereco        VARCHAR2(200) NOT NULL,
    cidade          VARCHAR2(100) NOT NULL,
    estado          VARCHAR2(50) NOT NULL,
    cep             VARCHAR2(10),
    area_total      NUMBER(10,2), -- em hectares
    usuario_id      NUMBER REFERENCES GS_WW_USUARIO(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Zonas de Irrigação
CREATE TABLE GS_WW_ZONA_IRRIGACAO (
    id              NUMBER PRIMARY KEY,
    nome            VARCHAR2(100) NOT NULL,
    area_m2         NUMBER(10,2) NOT NULL,
    tipo_cultura    VARCHAR2(50) NOT NULL,
    necessidade_agua NUMBER(5,2), -- litros por m²/dia
    status          VARCHAR2(20) DEFAULT 'ATIVA',
    propriedade_id  NUMBER REFERENCES GS_WW_PROPRIEDADE(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Sensores IoT
CREATE TABLE GS_WW_SENSOR (
    id              NUMBER PRIMARY KEY,
    codigo          VARCHAR2(50) UNIQUE NOT NULL,
    tipo            VARCHAR2(30) NOT NULL, -- UMIDADE, TEMPERATURA, LUMINOSIDADE
    localizacao     VARCHAR2(100),
    status          VARCHAR2(20) DEFAULT 'ATIVO',
    bateria_nivel   NUMBER(3,0), -- percentual 0-100
    zona_id         NUMBER REFERENCES GS_WW_ZONA_IRRIGACAO(id),
    installed_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Leituras dos Sensores
CREATE TABLE GS_WW_LEITURA_SENSOR (
    id              NUMBER PRIMARY KEY,
    valor           NUMBER(10,4) NOT NULL,
    unidade         VARCHAR2(10) NOT NULL, -- %, °C, lux
    timestamp       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    qualidade       VARCHAR2(20) DEFAULT 'NORMAL', -- NORMAL, SUSPEITA, ERRO
    sensor_id       NUMBER REFERENCES GS_WW_SENSOR(id),
    processed       CHAR(1) DEFAULT 'N',
    anomalia        CHAR(1) DEFAULT 'N'
);

-- Tabela de Eventos de Irrigação
CREATE TABLE GS_WW_IRRIGACAO (
    id              NUMBER PRIMARY KEY,
    inicio          TIMESTAMP NOT NULL,
    fim             TIMESTAMP,
    volume_litros   NUMBER(10,2),
    tipo_acionamento VARCHAR2(20) NOT NULL, -- AUTO, MANUAL, EMERGENCIA
    eficiencia      NUMBER(5,2), -- percentual de aproveitamento
    zona_id         NUMBER REFERENCES GS_WW_ZONA_IRRIGACAO(id),
    usuario_id      NUMBER REFERENCES GS_WW_USUARIO(id),
    observacoes     VARCHAR2(500)
);

-- Tabela de Alertas do Sistema
CREATE TABLE GS_WW_ALERTA (
    id              NUMBER PRIMARY KEY,
    tipo            VARCHAR2(30) NOT NULL, -- SOLO_SECO, SENSOR_OFFLINE, DESPERDICIO
    titulo          VARCHAR2(100) NOT NULL,
    mensagem        VARCHAR2(500) NOT NULL,
    prioridade      VARCHAR2(10) DEFAULT 'MEDIA', -- BAIXA, MEDIA, ALTA, CRITICA
    lido            CHAR(1) DEFAULT 'N',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    zona_id         NUMBER REFERENCES GS_WW_ZONA_IRRIGACAO(id),
    usuario_id      NUMBER REFERENCES GS_WW_USUARIO(id)
);

-- Tabela de Previsões de Irrigação (ML.NET)
CREATE TABLE GS_WW_PREVISAO_IRRIGACAO (
    id              NUMBER PRIMARY KEY,
    data_previsao   DATE NOT NULL,
    volume_estimado NUMBER(10,2) NOT NULL, -- litros
    confianca       NUMBER(5,2), -- percentual de confiança da IA
    fatores_clima   VARCHAR2(200), -- JSON com dados meteorológicos
    status          VARCHAR2(20) DEFAULT 'PENDENTE',
    zona_id         NUMBER REFERENCES GS_WW_ZONA_IRRIGACAO(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    accuracy_real   NUMBER(5,2) -- medido após evento real
);
```

### 🔧 **Package PL/SQL para Automação**

```sql
CREATE OR REPLACE PACKAGE PKG_GS_WW_SMART_IRRIGATION AS
    -- Procedimento para inserir leituras automáticas
    PROCEDURE proc_inserir_leitura(
        p_sensor_id IN NUMBER,
        p_valor IN NUMBER,
        p_unidade IN VARCHAR2
    );
    
    -- Função para calcular necessidade de irrigação
    FUNCTION func_calc_necessidade_irrigacao(
        p_zona_id IN NUMBER
    ) RETURN NUMBER;
    
    -- Procedure para gerar alerta automático
    PROCEDURE proc_gerar_alerta_automatico(
        p_zona_id IN NUMBER,
        p_tipo_alerta IN VARCHAR2
    );
    
    -- Cursor para relatório de eficiência mensal
    PROCEDURE proc_relatorio_eficiencia_mensal(
        p_propriedade_id IN NUMBER,
        p_mes IN NUMBER,
        p_ano IN NUMBER
    );
END PKG_GS_WW_SMART_IRRIGATION;
```

### 🍃 **Estrutura MongoDB (Dados Não-Relacionais)**

```javascript
// Coleção: system_logs
{
    "_id": ObjectId,
    "timestamp": ISODate,
    "level": "INFO|WARN|ERROR",
    "service": "api|sensor|ml|notification",
    "message": "string",
    "zona_id": NumberInt,
    "sensor_id": NumberInt,
    "metadata": {
        "ip_address": "string",
        "user_agent": "string",
        "response_time": NumberInt
    }
}

// Coleção: weather_predictions
{
    "_id": ObjectId,
    "date": ISODate,
    "location": {
        "propriedade_id": NumberInt,
        "coordinates": [longitude, latitude]
    },
    "forecast": {
        "temperature": { "min": Number, "max": Number },
        "humidity": Number,
        "precipitation": Number,
        "wind_speed": Number
    },
    "ml_analysis": {
        "irrigation_recommendation": "string",
        "confidence_score": Number,
        "model_version": "string"
    }
}

// Coleção: plant_images_metadata
{
    "_id": ObjectId,
    "zona_id": NumberInt,
    "image_url": "string",
    "captured_at": ISODate,
    "analysis": {
        "health_score": Number,
        "growth_stage": "string",
        "disease_detected": Boolean,
        "recommendations": ["string"]
    },
    "camera_info": {
        "device_id": "string",
        "resolution": "string",
        "location": [longitude, latitude]
    }
}
```

### 🎯 **Principais Melhorias Implementadas**

**1. Foco em Zonas de Irrigação**
- Cada zona tem área específica e tipo de cultura
- Sensores associados às zonas (não genéricos)
- Irrigação controlada por zona para precisão

**2. Controle de Eficiência**
- Campo `eficiencia` na tabela IRRIGACAO
- Tabela PREVISAO_IRRIGACAO com IA
- Campo `accuracy_real` para medir precisão das previsões

**3. Estrutura de Propriedade Completa**
- Endereço completo separado do usuário
- Área total para cálculos de proporção
- Múltiplas zonas por propriedade

**4. Alertas Inteligentes**
- Priorização de alertas (BAIXA a CRÍTICA)
- Alertas específicos por zona
- Tipos focados em economia de água

**5. Rastreabilidade Completa**
- Timestamps em todas as tabelas
- Status para controle de estados
- Campos de qualidade e anomalia

**6. Otimização para IoT**
- Campo `bateria_nivel` nos sensores
- Flag `processed` para processamento batch
- Código único para identificação de sensores

### 📊 **Métricas de Sustentabilidade Calculáveis**

- **Economia de Água**: Comparar consumo antes/depois
- **Eficiência por Zona**: Volume usado vs. necessidade real
- **Acurácia da IA**: Previsões vs. irrigações reais
- **Uptime dos Sensores**: Disponibilidade do sistema
- **ROI Ambiental**: Produtividade vs. consumo de água

Esta estrutura otimizada mantém o foco acadêmico enquanto demonstra conhecimento prático de sistemas IoT para agricultura sustentável! 🌱