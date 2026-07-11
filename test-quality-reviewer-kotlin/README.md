# gradle-llm-test-reviewer

Una tarea de Gradle que usa un LLM para revisar la **calidad** de los tests unitarios, no solo su cobertura. Detecta aserciones tautológicas, exceso de mocking, nombres de test poco descriptivos y casos límite ausentes — antes de que lleguen a una code review humana.

## El problema

Un test puede tener 100% de cobertura de línea y no proteger nada:

```kotlin
@Test
fun `should process refund`() {
    val service = RefundService(mockGateway, mockLedger)
    val result = service.processRefund(refundRequest)
    assertNotNull(result)
}
```

Este test "pasa" y "cubre" la línea, pero `processRefund` nunca devuelve `null` por firma, así que la aserción no puede fallar jamás. Herramientas de cobertura como JaCoCo no lo detectan. El mutation testing (PIT) sí lo detectaría, pero solo te dice *que* el mutante sobrevivió — no *por qué* el test es débil ni cómo arreglarlo.

## Qué hace

Por cada test modificado en un PR, la tarea:

1. Lee el archivo de test y la clase que prueba.
2. Se los pasa a un LLM (Claude, vía la API de Anthropic) junto con una rúbrica fija.
3. Recibe un veredicto en JSON: puntuación 0-100, lista de issues con severidad, y un resumen.
4. Falla el build si la puntuación cae por debajo de un umbral configurable.

## Ejemplo real

Contra la misma clase `RefundService`, comparamos un test débil y uno con tres casos de comportamiento (rechazo por importe negativo, éxito con verificación de ledger, fallo del gateway). Esta es la salida real de Claude, sin editar:

**Test débil → 8/100**

> "The single test provides almost no value: its only assertion is untriggerable, no behaviour or return-value properties are verified, collaborator interactions are unchecked, and none of the three distinct code paths (invalid amount, gateway failure, success) are deliberately targeted."

Issues de severidad alta: aserción tautológica, cero verificación de interacción con los mocks, nombre de test que no describe comportamiento, cero casos límite cubiertos.

**Test robusto → 72/100**

> "The three tests cover the main happy path and the two primary failure branches with appropriate mock usage and meaningful assertions, but miss the zero-amount boundary, a ledger-never-called assertion on rejection, transaction ID propagation on success, and gateway exception handling."

Lo interesante de este segundo caso: el modelo señaló que **ningún test cubre qué pasa si `gateway.refund()` lanza una excepción** en vez de devolver un fallo controlado — un caso que no estaba en la rúbrica explícita y que un revisor humano fácilmente pasaría por alto en una revisión rápida.

## Uso

### 1. Configura la API key

```bash
export ANTHROPIC_API_KEY="tu-key-aqui"
```

### 2. Ejecuta la revisión

```bash
./gradlew reviewTestQuality
```

Opcional: ajusta el umbral mínimo de puntuación (por defecto 70):

```bash
./gradlew reviewTestQuality -PtestReviewMinScore=60
```

### 3. Integra en CI

Añádelo como paso en tu pipeline, sobre los archivos de test modificados respecto a la rama destino:

```yaml
- name: Review test quality
  run: ./gradlew reviewTestQuality
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Cómo funciona por dentro

```
libs.versions.toml          →  build.gradle.kts          →  Prompt + rúbrica
(versiones centralizadas)      (task reviewTestQuality)     (src/test/resources/prompts/)
                                        │
                                        ▼
                              API de Anthropic (Claude)
                                        │
                                        ▼
                         JSON: { score, issues[], summary }
                                        │
                                        ▼
                          Falla el build si score < umbral
```

La rúbrica vive en `src/test/resources/prompts/test-quality-review.txt`, versionada junto al código que evalúa — no enterrada en el build script:

```
Score the test file from 0-100 on these criteria:
- Assertions are behavioural, not tautological
- Collaborators are mocked only where necessary
- Test names describe the specific behaviour under test
- Edge cases relevant to the method signature are covered
```

## Limitaciones

- **Es una heurística, no una verdad absoluta.** Puede juzgar mal un smoke test intencionadamente simple. Trátalo como una señal para revisar, no como una puerta que sustituye el juicio humano.
- **No reemplaza el mutation testing.** PIT te dice de forma objetiva si un mutante sobrevivió. Esta herramienta explica probables razones más rápido, pero son complementarias.
- **Coste y latencia.** Está pensado para correr solo sobre archivos de test modificados en un PR, no sobre toda la suite en cada commit.

## Stack

- Kotlin + Gradle Version Catalogs
- Spring Boot 4 (proyecto de ejemplo)
- API de Anthropic (Claude)

## Artículo relacionado

Este repo acompaña al artículo del blog de Parser: *"¿Puede la IA saber si tus tests son realmente buenos? Usando LLMs para revisar la calidad de los tests"*.