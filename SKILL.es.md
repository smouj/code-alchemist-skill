name: code-alchemist
version: 1.2.0
description: Motor automatizado de refactorización y transformación de arquitectura de código
author: SMOUJBOT
tags:
  - refactoring
  - code-quality
  - maintainability
  - architecture
  - cleanup
type: transformation
risk: medium
dependencies:
  - eslint
  - prettier
  - typescript
  - @typescript-eslint/parser
  - @typescript-eslint/eslint-plugin
  - biome
  - tsc
  - vitest
  - jest
required_tools:
  - node >= 18
  - git
  - jq
environment_vars:
  - REFACTOR_DRY_RUN: "Set to 'true' to preview changes without applying"
  - REFACTOR_AGGRESSIVE: "Set to 'true' for deeper transformations"
  - REFACTOR_SAFETY_LEVEL: "3 (max) to 1 (min) - controls how conservative changes are"
work_dir: /tmp/code-alchemist-{task_id}
timeout: 180000
```

# El Alquimista de Código

## Propósito

Transforma bases de código desordenadas, heredadas o mal estructuradas en arquitecturas limpias y mantenibles mediante refactorización automatizada. Dirigido a equipos que necesitan:
- Reducir deuda técnica en servicios Node.js/TypeScript en producción
- Estandarizar estilo y patrones de código en microservicios
- Migrar de JavaScript a TypeScript de forma incremental
- Extraer y separar responsabilidades (controladores pesados, objetos dios)
- Convertir código basado en callbacks a patrones async/await
- Eliminar code smells: funciones largas, lógica duplicada, anidamiento profundo
- Preparar bases de código para escalamiento de equipo mediante convenciones

**Casos de uso reales:**
- Limpieza pre-sprint: "Limpia nuestro servicio de autenticación antes de que se una el nuevo equipo"
- Migración: "Convierte todos los requires CommonJS a módulos ES de forma segura"
- Corrección de arquitectura: "Divide este servicio de 2000 líneas en módulos de dominio"
- Puerta de calidad: "Asegura que todos los nuevos PRs cumplan nuestros estándares de linting"

## Alcance

Ejecuta dentro de un repositorio objetivo para realizar transformaciones controladas y reversibles:

### Comandos
- `analyze`: Análisis profundo de la base de código, identifica puntos críticos y candidatos a refactorización
- `refactor:format`: Aplica formato consistente (Prettier/Biome)
- `refactor:lint`: Corrige problemas de lint automáticamente
- `refactor:typescript`: Migración incremental a TypeScript, añade tipos faltantes
- `refactor:extract`: Extrae código duplicado en utilidades
- `refactor:layers`: Reorganiza en capas de arquitectura limpia
- `validate:quality`: Verificación de calidad - asegura que el código cumple estándares
- `rollback`: Revierte última transformación usando git stash

### Flags
- `--dry-run`: Previsualiza sin modificar archivos
- `--aggressive`: Habilita transformaciones más profundas y riesgosas
- `--scope=<glob>`: Limita a directorios específicos (por defecto: repositorio completo)
- `--exclude=<glob>`: Omite ciertos patrones (node_modules, tests, migraciones)
- `--safety-level=N`: Conservador (1) a agresivo (5)
- `--commit`: Crea commit git después del refactor exitoso
- `--skip-tests`: No ejecutar suite de pruebas (usar con precaución)

## Proceso de Trabajo

### 1. Descubrimiento y Línea Base
```bash
# Clona objetivo en espacio aislado
git clone --depth 1 <repo_url> /tmp/code-alchemist-{uuid}

# Captura métricas base
./node_modules/.bin/eslint . --format json > baseline-eslint.json
./node_modules/.bin/typescript --noEmit > baseline-tsc.txt
find src -name "*.ts" -o -name "*.js" | wc -l > file-count.txt
```

**Validación:** Asegura que el repositorio esté sano (build exitoso, tests pasan) antes de refactorizar.

### 2. Fase de Análisis
```bash
# Ejecuta linter con detección de autofix
npx eslint src/ --fix-dry-run --format json > fixable-issues.json

# Identifica code smells
grep -r "any" src/ --include="*.ts" > any-usage.txt
grep -r "console.log" src/ --include="*.ts" > debug-statements.txt
find src -name "*.ts" -exec wc -l {} + | sort -rn | head -20 > longest-files.txt
```

**Salida:** `analysis-report.json` con:
- Total de issues por categoría
- Top 10 archivos que necesitan atención
- Tiempo de refactor estimado

### 3. Pipeline de Transformación
Ejecuta secuencialmente, deteniéndose en fallo:

**Paso A: Formateo**
```bash
npx biome format --write src/
# O fallback a prettier
npx prettier --write src/
```

**Paso B: Correcciones de Lint**
```bash
npx eslint src/ --fix
```

**Paso C: Mejoras TypeScript**
```bash
# Añade exports/imports faltantes
npx tsc --noEmit

# Tipado incremental: convierte any implícito a unknown primero
# Script personalizado: scripts/improve-types.mjs
node scripts/improve-types.mjs --strategy conservative
```

**Paso D: Extracción de Utilidades**
- Detecta bloques de código duplicado (similitud 40+ líneas)
- Crea plan de extracción
- Mueve a `src/shared/` o `src/utils/`
- Actualiza todos los imports

**Paso E: Separación de Capas**
Si flag `--layers`:
```
src/
├── domain/     # Lógica de negocio, entidades
├── application/ # Casos de uso, servicios
├── infrastructure/ # DB, APIs externas
└── presentation/ # Controladores, rutas, UI
```

### 4. Verificación
```bash
# Type check
npx tsc --noEmit

# Lint
npx eslint src/

# Tests (a menos que --skip-tests)
npm test

# Build verification
npm run build
```

Todos deben pasar. Si alguno falla, auto-rollback a línea base.

### 5. Commit y Documentación
```bash
git add .
git commit -m "refactor: automated cleanup by Code Alchemist

- Format: Biome
- Lint: ESLint fixes
- Types: Incremental TypeScript improvements
- Extract: Shared utilities from duplication
- Layers: Reorganized into clean architecture

Stats:
- Files changed: X
- Lines added: Y
- Lines removed: Z
- Estimated debt reduction: [from analysis]"
```

**Log:** `refactor-log.md` con métricas antes/después.

## Reglas de Oro

1. **Nunca romper el build.** Si typecheck o tests fallan tras cualquier paso, rollback inmediato.
2. **Preservar comportamiento.** Los cambios de refactorización alteran solo estructura, sin cambios de lógica.
3. **Commits pequeños.** Cada transformación crea su propio commit (o grupo), nunca cambios monolíticos.
4. **Tests primero.** Verifica que los tests pasan antes de empezar; ejecuta tests tras cada paso mayor.
5. **Respetar configuraciones del proyecto.** Usa configuraciones ESLint/Prettier/TypeScript existentes; no imponer reglas externas.
6. **No guerras de formato.** Mantén un solo formateador (Biome preferido, luego Prettier, luego ESLint).
7. **Evitar sobre-ingeniería.** No introduzcas patrones que la base de código no necesite aún.
8. **Mantener tests en verde.** Si `--skip-tests` no está activo, tests fallando = abortar.
9. **Documentar extracción.** Al extraer utilidades, añade JSDoc explicando propósito.
10. **Respetar límites.** No cruces límites de microservicios; refactoriza solo dentro del repositorio.

## Ejemplos

### Ejemplo 1: Limpieza básica
**Prompt del usuario:**
```
code-alchemist refactor:format --scope=src --dry-run
```

**Ejecución de Skill:**
```bash
cd /tmp/code-alchemist-task123
npx biome format --write src/
# Dry run: npx biome format src/ --print-diff > format-changes.diff
```

**Salida:**
```
Formateando 47 archivos en src/
- Reformateado: src/auth/AuthService.ts
- Reformateado: src/users/UserController.ts
- Omitido: src/vendor/ (patrón excluido)

Diff de dry-run guardado en: /tmp/format-changes.diff
```

### Ejemplo 2: Pipeline completo de transformación
**Prompt del usuario:**
```
code-alchemist transform --aggressive --commit --safety-level=4
```

**Ejecución de Skill:**
```bash
# 1. Analizar
node scripts/analyze.mjs --output analysis.json
# analysis.json: {"hotspots": 3, "duplication": 12%, "any-usage": 45}

# 2. Formatear
npx prettier --write .

# 3. Lint fix
npx eslint . --fix

# 4. Mejoras de tipos
node scripts/improve-types.mjs --strategy aggressive
# Cambios: 23 any → unknown, añadidos 15 returns explícitos

# 5. Extraer duplicados
node scripts/extract-duplicates.mjs --threshold 40
# Creado: src/shared/pagination.ts, src/shared/validation.ts

# 6. Verificar
npm test && npm run build

# 7. Commit
git add .
git commit -m "refactor: comprehensive cleanup (Code Alchemist automated)"
```

**Log de salida:**
```
[SUCCESS] Transformación completa
- Archivos procesados: 148
- Líneas eliminadas: 234
- Issues de tipos corregidos: 67
- Duplicados extraídos: 2 módulos
- Estado de tests: 42 pasaron, 0 fallaron
- Commit: abc1234
```

### Ejemplo 3: Migración TypeScript
**Prompt del usuario:**
```
code-alchemist refactor:typescript --target=js-files --strategy=incremental
```

**Ejecución de Skill:**
```bash
# Encuentra archivos .js sin tests primero
find src -name "*.js" ! -path "*/test/*" > js-files.txt

# Renombra a .ts, corrige imports, añade tipos any básicos
node scripts/migrate-js-to-ts.mjs --file-list js-files.txt --add-any
# Output: Convertidos 8 archivos a .ts
```

**Salida:**
```
Migrando:
- src/utils/helpers.js → src/utils/helpers.ts
- src/middleware/auth.js → src/middleware/auth.ts

Siguientes pasos:
1. Revisar tipos 'any' añadidos (intencionales para migración)
2. Ejecutar: code-alchemist refactor:typescript --improve-types
3. Reemplazar gradualmente any por tipos específicos
```

## Comandos de Rollback

### Rollback manual (última transformación)
```bash
# Si se hizo commit con --commit
git revert HEAD --no-edit
git push origin <branch>

# Si no hay commit pero existe stash
git stash list | grep "Code Alchemist"
git stash pop stash@{0}
```

### Rollback proporcionado por Skill
```bash
code-alchemist rollback --last
# Equivalente a: git reset --hard ORIG_HEAD
```

### Reset completo de repositorio (peligroso)
```bash
code-alchemist rollback --to-baseline --commit-hash=<hash>
# Crea nuevo commit que restaura cada archivo al estado base
```

### Verificar seguridad de rollback
```bash
# Ver qué se revertiría
code-alchemist rollback --preview
# Muestra: Archivos a restaurar, líneas a eliminar/añadir
```

## Pasos de Verificación

Después de cualquier operación de refactor, ejecuta:

```bash
# 1. Type check
npx tsc --noEmit
# Esperado: Sin errores (warnings OK si pre-existentes)

# 2. Lint
npx eslint src/ --max-warnings 0
# Esperado: 0 warnings

# 3. Tests
npm test -- --coverage
# Esperado: 100% pasa, cobertura >= base

# 4. Tamaño de bundle
npm run build && du -sh dist/
# Esperado: Sin aumento significativo (>5%)

# 5. Smoke test de runtime
node dist/index.js --smoke-test
# Esperado: "OK" dentro de 2 segundos

# 6. Métricas de código
node scripts/measure-complexity.mjs
# Esperado: Complejidad ciclomática ↓, índice mantenibilidad ↑
```

Documenta resultados en `refactor-audit.md` con:
- Conteo LOC antes/después
- Conteo de issues (eslint, tsc)
- Tasa de paso de tests
- Cualquier ítem de revisión manual

## Solución de Problemas

### Issue: "Refactor rompió los tests"
**Fix:** Rollback automático activado al fallo de tests. Si persiste:
```bash
# Examina cambios en archivo de test fallido
git diff HEAD~1 src/tests/MyTest.ts
# Ajusta manualmente refactor para preservar comportamientos de test
code-alchemist refactor:format --exclude="src/tests/**"
```

### Issue: "No puedo corregir regla de lint XYZ"
**Causa:** Regla requiere intervención manual (seguridad, mejor práctica).
**Fix:**
```bash
npx eslint src/ --rule '{"no-eval": "off"}' --fix
# O excluir patrón de archivo específico:
code-alchemist refactor:lint --exclude="src/legacy/**"
```

### Issue: "Errores TypeScript aumentaron tras migración"
**Fix:** Ejecuta mejora en fases:
```bash
# Fase 1: Añadir tipos any (ya hecho)
# Fase 2: Reemplazar any por unknown
node scripts/improve-types.mjs --strategy replace-any-unknown
# Fase 3: Revisión manual de cualquier any restante
grep -r ": any" src/
```

### Issue: "Conflictos de formato con Prettier"
**Fix:** Detecta formateador del proyecto:
```bash
if [ -f .prettierrc ]; then
  FORMATTER="prettier"
elif [ -f biome.json ]; then
  FORMATTER="biome"
else
  FORMATTER="eslint --fix"
fi
```

Set `REFACTOR_FORMATTER=$FORMATTER` environment variable.

### Issue: "Memoria insuficiente durante análisis"
**Fix:** Procesa archivos en lotes:
```bash
code-alchemist analyze --batch-size=50 --concurrency=4
```

### Issue: "Conflictos git durante rollback"
**Fix:** Usa estrategia de merge:
```bash
git merge --abort
git reset --hard ORIGIN/$(git branch --show-current)
```

## Dependencias

**Instalación:**
```bash
npm install --save-dev \
  eslint \
  prettier \
  @typescript-eslint/parser \
  @typescript-eslint/eslint-plugin \
  biome \
  typescript \
  vitest \
  jest
```

**Archivos de configuración requeridos:**
- `.eslintrc.js` (o .json/.yaml)
- `.prettierrc` (opcional)
- `biome.json` (opcional)
- `tsconfig.json`
- `package.json` con scripts: `test`, `build`, `lint`

**¿Sin dependencias?** Skill fallará con mensaje claro:
```
ERROR: No se encontró config de ESLint. Inicializa con: npx eslint --init
```

## Características de Seguridad

- Dry-run por defecto para operaciones destructivas
- Stashing automático antes de cambios
- Pre-flight checks: repositorio limpio? tests pasando?
- Verificación post-flight obligatoria
- Punto de rollback creado antes de cada fase
- Sin cambios si riesgo excede umbral `--safety-level`

```