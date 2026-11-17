# Laboratorio: pipeline de CI con GitHub Actions en una API Laravel

Curso IS-803, grupo 04.

## Propósito

Demostrar cómo un pipeline de integración continua (CI) en GitHub Actions
verifica automáticamente un backend Laravel en cada `push` y `pull_request`:
instala dependencias, valida la sintaxis, prepara el entorno y ejecuta las
pruebas. El proyecto expone un único endpoint, `GET /api/health`, con una
prueba de característica asociada.

## Requisitos previos

- Git instalado.
- Cuenta de GitHub.
- PHP 8.3 o superior.
- Composer 2.

## Pasos guiados

### 1. Crear o usar un repositorio Git del backend, sin `.env` ni claves

```bash
git clone https://github.com/<usuario>/<repositorio>.git
cd <repositorio>
```

Si se parte de este proyecto sin remoto:

```bash
git remote add origin https://github.com/<usuario>/<repositorio>.git
```

Verificar que `.env` está ignorado y no versionado:

```bash
git check-ignore .env      # debe imprimir ".env"
git ls-files | grep -c '^\.env$'   # debe imprimir 0
```

### 2. Lograr que las pruebas pasen en local

```bash
composer install
cp .env.example .env
php artisan key:generate
php artisan test
```

El resultado esperado es una prueba en verde (`HealthTest`). Las pruebas
usan SQLite en memoria (definido en `phpunit.xml`), por lo que no requieren
una base de datos real.

### 3. Crear el workflow `.github/workflows/ci.yml`

El archivo debe estar exactamente en `.github/workflows/` y declarar los
disparadores `push` y `pull_request`:

```yaml
on: [push, pull_request]
```

### 4. Definir el job en ubuntu-latest

El job `verificar` se ejecuta en `ubuntu-latest` y realiza, en orden:

1. Obtener el código (`actions/checkout@v4`).
2. Configurar PHP 8.3 (`shivammathur/setup-php@v2`).
3. Instalar dependencias con Composer.
4. Validar la sintaxis de los archivos PHP.
5. Copiar `.env.example` a `.env` y generar `APP_KEY`.
6. Ejecutar `php artisan test`.

### 5. Subir un cambio y revisar la pestaña Actions

```bash
git add .
git commit -m "ci: add GitHub Actions workflow"
git push -u origin main
```

En GitHub, abrir la pestaña **Actions** del repositorio, seleccionar la
ejecución más reciente de "CI Laravel" y revisar el estado del job
`verificar` y el log de cada paso.

### 6. Provocar un fallo controlado, corregirlo y volver a verde

En `tests/Feature/HealthTest.php`, cambiar la aserción:

```php
$response->assertJson(['status' => 'fallo']);
```

Confirmar el fallo en local y publicarlo:

```bash
php artisan test            # termina con código distinto de cero
git commit -am "test: break health assertion on purpose"
git push
```

El pipeline aparecerá en rojo en la pestaña **Actions**; revisar el log del
paso "Ejecutar pruebas". Para corregirlo, restaurar `'ok'`:

```bash
git commit -am "fix: restore health assertion"
git push
```

El pipeline vuelve a verde. También puede revertirse con
`git revert HEAD`.

## Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `composer install` falla en CI con "requires php >= 8.4" | `composer.lock` generado con una versión de PHP local más nueva que la del CI | Mantener `config.platform.php` en `composer.json` (aquí `8.3.0`) y ejecutar `composer update` para regenerar el lock |
| "No application encryption key has been specified" | Falta `.env` o `APP_KEY` | Ejecutar `cp .env.example .env` y `php artisan key:generate` |
| El workflow no aparece en Actions o marca error de sintaxis | Archivo fuera de `.github/workflows/` o indentación YAML incorrecta (usar espacios, no tabulaciones) | Verificar la ruta exacta y validar la indentación |
| Las pruebas pasan en local pero fallan en CI | Diferencias de versión de PHP, dependencia de `.env` local, de una base de datos o de archivos no versionados | Reproducir con la misma versión de PHP, clonar el repositorio en limpio y revisar el log del paso fallido |

## Lista de verificación de seguridad

- [ ] `.env` no está versionado (figura en `.gitignore`).
- [ ] No hay claves, tokens ni contraseñas en el repositorio ni en `.env.example`.
- [ ] Se revisan los logs de Actions antes de compartirlos o hacerlos públicos.
- [ ] Las credenciales reales se almacenan en GitHub Secrets
      (Settings > Secrets and variables > Actions), nunca en el código.
