# MODGAMES AMPR LZ4

**Versión estable de referencia: v1.0**

<img width="1278" height="934" alt="Captura de pantalla 2026-09-26 163853" src="https://github.com/user-attachments/assets/2b0c73cf-4bfa-4eac-90a9-9623b4c034c0" />


MODGAMES AMPR LZ4 es una herramienta de escritorio para **preparar, analizar, empaquetar y verificar contenido LZ4 basado en AMPR** para juegos PS5.

El proyecto está orientado a un flujo de trabajo reproducible en Windows, manteniendo separados el análisis del contenido, la generación del perfil y la construcción de los packs LZ4.

> **Importante:** v1.0 es la rama estable de referencia del flujo LZ4. Las funciones experimentales de análisis automático desarrolladas después de v1.0 no forman parte de esta versión.

---

## Características

### Trace Analyzer

Permite analizar trazas AMPR y convertir la información observada durante la ejecución del juego en un perfil TOML.

El perfil puede describir:

- archivos y rutas observadas;
- tamaño de bloque;
- modo de acceso;
- clasificación de los recursos;
- grupos de packs;
- parámetros de runtime;
- recomendaciones de caché y workers;
- cobertura observada por las trazas.

El perfil generado debe considerarse **un punto de partida**, no una prueba de cobertura completa: una traza sólo representa las lecturas observadas durante los escenarios registrados. El propio generador de perfiles recomienda revisar niveles, modos, idiomas y DLC que no hayan sido incluidos en la captura.

### LZ4 PACK

Construye los packs AMPR a partir de:

- `ampr_emu.index`;
- un perfil TOML;
- la carpeta del juego;
- los parámetros de empaquetado definidos en el perfil.

El backend utiliza LZ4 por bloques y admite perfiles con distintos tamaños, layouts, grupos y políticas de compresión.

### Verificación

Después de construir los packs, el flujo de trabajo permite ejecutar la verificación de integridad de los datos generados.

La metodología recomendada es:

```text
Construir
   ↓
Verificar
   ↓
Probar manteniendo los archivos originales disponibles
```

### Archivos originales conservados

El flujo está diseñado para trabajar de forma conservadora con los archivos del juego. Las protecciones de perfil mantienen fuera del pack los elementos sensibles, incluidos ejecutables, módulos PRX/SPRX/ELF, índices AMPR y otras rutas protegidas.

Ejemplo de protecciones de la versión de referencia:

```text
sce_module/**
system/**
mods/**
save/**
**/*.prx
**/*.sprx
eboot.bin
ampr_emu.index
ampr_assets.index
ampr_assets.index.crc
ampr_assets.index.runtime
ampr_assets-*.pak
```

---

## Arquitectura

```text
                  ┌──────────────────────┐
                  │    CARPETA DEL JUEGO │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │    TRACE ANALYZER    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │       PERFIL TOML    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │     ampr_pack.py     │
                  │      LZ4 PACK        │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │   AMPR LZ4 PACKS     │
                  │   + índice/checks    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │       VERIFY         │
                  └──────────────────────┘
```

La aplicación GUI de MODGAMES funciona como capa de coordinación sobre las herramientas AMPR, manteniendo el análisis y el empaquetado como etapas separadas.

---

## Conceptos principales

### `ampr_emu.index`

Índice que describe los archivos disponibles para el runtime AMPR.

### Perfil TOML

El perfil determina qué archivos se consideran `compress`, `loose` u otras variantes admitidas por el packer, además de sus parámetros.

Un perfil de referencia puede comenzar con:

```toml
[pack]
index_name = "ampr_assets.index"
pack_pattern = "ampr_assets-{id:03d}.pak"
default_action = "loose"
default_block_size = "64KiB"
compression_mode = "hc"
compression_level = 12
```

El uso de `default_action = "loose"` permite mantener un comportamiento conservador y añadir explícitamente los recursos que deben entrar en los packs.

### Reglas

El packer resuelve las reglas siguiendo **last match wins**: una regla posterior puede reemplazar la decisión de una regla anterior.

Esto permite utilizar reglas generales seguidas por excepciones más específicas.

### `force_pack`

Las reglas pueden utilizar `force_pack = true` cuando se desea evitar que la heurística de `auto-loose` convierta un archivo grande en `loose`.

---

## Flujo de trabajo recomendado

### 1. Preparar el juego

Partir de una copia de trabajo de la carpeta del juego.

Es recomendable conservar siempre una copia limpia del contenido original.

### 2. Preparar el índice AMPR

Generar o disponer de:

```text
ampr_emu.index
```

### 3. Capturar una traza

Cuando el juego sea compatible con el flujo de tracing, registrar las situaciones que interesen:

- arranque;
- menú;
- carga inicial;
- niveles;
- transiciones;
- contenido que se quiera estudiar.

Una traza no garantiza que se hayan observado todos los recursos del juego.

### 4. Generar o editar el TOML

Revisar el perfil generado y comprobar especialmente:

- archivos incluidos;
- carpetas incluidas;
- archivos protegidos;
- tamaño de bloque;
- layout;
- grupos;
- cachés;
- workers;
- archivos que deben permanecer `loose`.

### 5. Construir

Conceptualmente:

```powershell
python ampr_pack.py pack `
  --root "<APP0>" `
  --ampr-index "<APP0>\ampr_emu.index" `
  --output "<OUTPUT>" `
  --config "<PROFILE>.toml"
```

Los argumentos exactos pueden variar según el wrapper o la configuración de la versión instalada; el GUI de MODGAMES prepara y ejecuta el flujo por el usuario.

### 6. Verificar

Ejecutar la verificación del pack generado antes de probarlo en el entorno final.

### 7. Probar con los originales disponibles

Durante las primeras pruebas, mantener los archivos originales accesibles permite distinguir un problema del pack frente a un problema de cobertura del perfil.

---

## Compresión LZ4

El backend utiliza bloques LZ4.

El perfil permite seleccionar:

- `fast`
- `hc`

y un nivel de compresión.

En los perfiles de referencia se utiliza:

```toml
compression_mode = "hc"
compression_level = 12
```

La elección del nivel debe considerarse un parámetro de construcción, no una garantía de compatibilidad del juego.

---

## Caché y runtime

El perfil también controla parámetros del runtime AMPR, por ejemplo:

```toml
[runtime]
decoded_cache_bytes = "64MiB"
physical_cache_bytes = "32MiB"
workers = 4
latency_reserve_workers = 0
```

Los perfiles generados a partir de trazas pueden proponer valores diferentes según el comportamiento observado. Por eso conviene conservar esos valores cuando procedan de una medición concreta y no mezclarlos con valores de otro juego sin validación.

---

#

## Diseño conservador

MODGAMES AMPR LZ4 está pensado para que el usuario pueda decidir exactamente qué contenido entra en los packs.

El enfoque de referencia es:

```text
default = LOOSE
       ↓
selección explícita
       ↓
protecciones
       ↓
pack
       ↓
verify
```

Esto evita asumir que un archivo es seguro de interceptar sólo porque tenga una buena relación de compresión.

Las trazas también deben interpretarse como evidencia parcial: representan sólo las lecturas observadas en los escenarios registrados.

---

## Limitaciones conocidas

### Las trazas no representan todo el juego

Una captura de un escenario no prueba que todas las rutas de todos los niveles, modos, idiomas y DLC hayan sido observadas.

### La compresión no equivale a compatibilidad

Que un archivo comprima bien no significa automáticamente que sea un buen candidato para acceso mediante AMPR.

### Los perfiles son específicos por contenido

Un perfil que funciona correctamente para un juego no debe asumirse como válido para otro.

### Las modificaciones del juego invalidan perfiles anteriores

Si cambia:

- la estructura de carpetas;
- los assets;
- el executable;
- el contenido instalado;
- DLC;
- idiomas;
- versión del juego;

se recomienda volver a revisar o regenerar el perfil.

---

## Filosofía del proyecto

MODGAMES AMPR LZ4 separa tres problemas:

```text
1. ANALIZAR
   ¿Qué archivos utiliza el juego?

2. EMPAQUETAR
   ¿Cómo convertir esos recursos en packs LZ4?

3. VALIDAR
   ¿El resultado generado es íntegro y reproducible?
```

El Trace Analyzer ayuda con el primer problema.

`ampr_pack.py` se ocupa del segundo.

La verificación y las herramientas de análisis ayudan con el tercero.

Esta separación permite mejorar el analizador sin tener que reemplazar el motor de empaquetado.

---

## Estado de la v3.46

**v3.46 — LZ4-only / stable reference**

Esta versión es la base estable del proyecto LZ4 antes de los experimentos posteriores de análisis automático.

Las funciones experimentales de perfiles `AUTO` desarrolladas después deben mantenerse fuera de esta release hasta que sean validadas con juegos reales.

---

## Créditos y dependencias

MODGAMES AMPR LZ4 integra herramientas y componentes del ecosistema AMPR/LZ4.

Cada dependencia debe conservar sus respectivas licencias y avisos originales cuando corresponda.

Antes de publicar una distribución binaria, revisar el directorio de licencias y las condiciones de cada dependencia incluida.

---

## Roadmap

### v3.46

- [x] GUI MODGAMES
- [x] Trace Analyzer
- [x] perfiles TOML
- [x] LZ4 PACK
- [x] verificación
- [x] flujo de entrega
- [x] build Windows

### Futuro

- [ ] análisis estático mejorado sin trace
- [ ] detección de motores/frameworks
- [ ] predicción de candidatos LZ4
- [ ] herramientas adicionales de diagnóstico
- [ ] mejoras de visualización de métricas de pack
- [ ] soporte experimental para nuevos perfiles y heurísticas

> Las funciones futuras deben mantenerse separadas del flujo estable hasta que hayan sido validadas con juegos reales.

---

## Licencia

La licencia final del proyecto debe ser definida por el mantenedor de MODGAMES antes de publicar el repositorio.

No se debe asumir una licencia para componentes de terceros que no la indiquen explícitamente.

---

## Aviso

Este proyecto está orientado al desarrollo, análisis y experimentación con sistemas de empaquetado y compresión LZ4/AMPR.

La compatibilidad real depende del juego, del contenido, del runtime utilizado y de la configuración del perfil. Un perfil válido para un título no constituye una garantía para otro.
