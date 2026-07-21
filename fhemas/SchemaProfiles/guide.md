# SchemaProfile Build Guide

## 1.Propósito y Alcance

Un **SchemaProfile** es un archivo EDN declarativo que instruye al parser de Fhemas sobre cómo agrupar lógicamente y extraer campos de un **StructureDefinition**.

Su alcance es exclusivo: está diseñado únicamente para definir las reglas de parsing de StructureDefinitions, no de otros tipos de recursos FHIR.

## 2.Estructura de Nivel Raíz

Los campos marcados como **Obligatorios** deben estar presentes para que el schema de Malli definido en Fhemas valide el recurso.

| Campo            | Tipo      | Obligatorio | Descripción                                                                                                          |
| ---------------- | --------- | :---------: | -------------------------------------------------------------------------------------------------------------------- |
| `:resource-type` | `string`  |   **Sí**    | Debe ser exactamente `"SchemaProfile"`.                                                                              |
| `:url`           | `string`  |   **Sí**    | URL canónica y resoluble del perfil.                                                                                 |
| `:version`       | `string`  |   **Sí**    | Versión semántica del perfil (ej. `"1.0.0"`).                                                                        |
| `:status`        | `keyword` |   **Sí**    | Estado de publicación: `:active`, `:draft`, `:retired` o `:unknown`.                                                 |
| `:fhir-version`  | `string`  |   **Sí**    | Versión de FHIR a la que aplica este perfil (ej. `"4.0.1"`).                                                         |
| `:source`        | `string`  |   **Sí**    | URL a la documentación oficial de la especificación FHIR a partir de la cual se creó este recurso.                   |
| `:schema`        | `map`     |   **Sí**    | Contiene las agrupaciones lógicas y reglas de extracción.                                                            |
| `:id`            | `string`  |     No      | Identificador único del perfil. En caso de no estar el motor se encargará de generarlo, pero se recomienda que esté. |
| `:title`         | `string`  |     No      | Título legible para humanos.                                                                                         |
| `:description`   | `string`  |     No      | Descripción del propósito del perfil.                                                                                |

## 3.El Bloque `:schema` y sus Agrupaciones Lógicas

La clave `:schema` divide el proceso de extracción en cuatro agrupaciones lógicas. El parser itera sobre estos vectores para saber qué campos buscar y cómo procesarlos.

1. **`:meta`**: Agrupa la metadata general que describe al recurso que se está modelando (ej. `:id`, `:url`, `:name`, `:version`, `:status`).
2. **`:definition`**: Agrupa las propiedades fundamentales que definen la esencia y el comportamiento del recurso (ej. `:kind`, `:abstract`, `:base-definition`, `:derivation`).
3. **`:invariants`**: Agrupa las restricciones y reglas de negocio, típicamente expresadas en FHIRPath.
4. **`:elements`**: Define dónde y cómo se extraen los elementos estructurales del recurso. **Esta es la única sección con una estructura anidada especial** (ver sección 5).

## 4.Definición de un Campo (`Field`)

Cada elemento dentro de los vectores de las agrupaciones lógicas es un mapa que define cómo extraer un dato.

```clojure
{:path [:nombre-del-campo]      ;; OBLIGATORIO
 :type :string                  ;; OPCIONAL
 :min 1                         ;; OPCIONAL
 :max 1                         ;; OPCIONAL
 :constraint {...}              ;; OPCIONAL
 :compile-as :ns/funcion}       ;; OPCIONAL
```

### Reglas Estrictas de los Campos

- **`:path` (Obligatorio)**: Indica la ruta de extracción. Puede ser:
  - Un vector de keywords: `[:url]`
  - Un mapa con expresión regular para coincidencia de patrones: `{:re-str "^fixed-.*$"}`
- **Convención de Omisión para `:min` y `:max`**:
  - Si la cardinalidad mínima es `0`, **no se incluye** la clave `:min`.
  - Si la cardinalidad máxima es ilimitada (`"*"`), **no se incluye** la clave `:max`.
  - _Regla de validación_: Si `:min` o `:max` están presentes en el EDN, su valor debe ser un entero `>= 1`.
- **`:type`**: Puede ser un keyword simple (`:string`) o un vector de keywords para tipos polimórficos (`[:string :integer :boolean]`).
- **`:compile-as`**: Debe ser un **keyword calificado** (con namespace, ej. `:fhemas.compile.r4/slicing`). Indica al parser que debe pasar el elemento completo a una función específica de Clojure para su transformación.
- **`:constraint`**: Es un mapa permisivo (`[:map-of :keyword :any]`) para permitir proyecciones de datos (ej. renombrar claves de FHIRPath a nombres internos) y futuras extensiones sin romper el schema.

## 5. La Sección Especial `:elements`

A diferencia de las otras tres agrupaciones, `:elements` tiene una estructura anidada porque debe manejar la dualidad de las vistas de un StructureDefinition.

```clojure
:elements
{:snapshot {:path [:snapshot :element]}
 :differential {:path [:differential :element]}
 :fields
 [{:path [:path] :type :string :min 1 :max 1}
  ;; ... resto de las definiciones de campo
  ]}
```

### Lógica del Parser para `:elements`

1. El parser busca primero en la ruta definida en `:snapshot`. Si existe, se considera la **vista completa** y lista para usar.
2. Si `:snapshot` no existe, el parser busca en la ruta definida en `:differential`.
3. Si solo se encuentra `:differential`, el parser sabe que debe activar su lógica de **resolución recursiva**: buscar el recurso padre (vía `:base-definition`), obtener su vista completa y aplicar los cambios del differential encima.
4. Una vez resuelta la fuente de datos (ya sea snapshot directo o differential resuelto), el parser aplica las reglas definidas en el vector `:fields` a cada elemento extraído.
