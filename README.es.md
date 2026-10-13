# Método de testing e2e agéntico

Versión corta en castellano. El método completo está en inglés: [README](README.md), [METHOD](METHOD.md) y [DECISIONS](DECISIONS.md).

**Qué es.** Un método para pruebas end-to-end donde un agente de IA propone qué probar, una persona lo aprueba, Playwright lo ejecuta, y una capa de juicio aparte decide si el resultado está bien.

**La regla que lo sostiene:** ningún componente valida su propio output. La intención la fija una persona. Lo aprobado no cambia a escondidas al pasar a código ni al correr. Quien propuso o ejecutó un test nunca da el veredicto.

**Las tres piezas.**

1. Un agente planificador lee un cambio y propone escenarios en Gherkin, para que una persona los apruebe.
2. Playwright ejecuta los tests aprobados y congelados, y junta la evidencia.
3. Un juez, otro modelo que no ve el razonamiento del agente, califica cada criterio contra una versión fija de criterios, siempre con evidencia.

**Para quién.** Equipos de QA y e2e que ya usan Playwright y quieren sumar un agente sin entregarle el veredicto.

**Estado.** Método v0.2, borrador. Todavía sin código. El harness se va a construir contra [Ghost](https://github.com/TryGhost/Ghost) y se va a demostrar sobre dos o tres pull requests reales.

**Relacionado.** El [observatorio de chatbots de soporte](https://github.com/missyeahhh/chatbot-observatory): la misma idea, criterios medibles y evidencia obligatoria, aplicada a otro sistema.

Licencia [CC BY 4.0](LICENSE).
