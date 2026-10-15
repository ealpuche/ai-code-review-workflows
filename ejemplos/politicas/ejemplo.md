Describe en una o dos líneas qué es este repositorio y quién lo usa. Indica el idioma de los comentarios.

Revisa solo defectos introducidos por este PR. Prioriza:

1. Correctitud: errores de lógica con un camino de ejecución concreto.
2. Integridad de datos: duplicados, idempotencia, nulos, compatibilidad de esquemas.
3. Seguridad: secretos, entrada no confiable, permisos excesivos.
4. Costo operativo: consultas, bucles o llamadas de red que puedan crecer sin control.
5. Pruebas faltantes para el comportamiento que cambió.

No comentes:

- Estilo que ya revisan los linters.
- Refactors sin un defecto observable.
- Hallazgos especulativos sin un camino de ejecución concreto.
- La misma causa raíz en varios comentarios.

Cada hallazgo lleva severidad, evidencia, impacto y una corrección concreta. Si no hay hallazgos materiales, dilo claramente.
