# Sistema Agéntico de Conciliación Bancaria

Pre-entrega 1 del proyecto integrador de AI Automation Avanzado.

Esta primera versión implementa un agente base en n8n para el análisis preliminar de casos de conciliación bancaria.

## Componentes

- Chat Trigger
- AI Agent
- Google Gemini Chat Model
- Google Sheets como Tool para consultar reglas de conciliación
- System Prompt con guardrails
- Límite de 5 iteraciones
- Gmail como log de observabilidad

## Objetivo

Analizar casos preliminares de conciliación bancaria, consultar reglas de negocio y clasificar situaciones como:

- CONCILIADO
- CONCILIADO CON DIFERENCIA
- REVISAR
- BANCO SIN SISTEMA
- SISTEMA SIN BANCO
- NO CONCILIABLE

Este workflow corresponde al Módulo 1 y será ampliado en los siguientes checkpoints con arquitectura multi-agente, memoria, integraciones, RAG, supervisión y trazabilidad.
