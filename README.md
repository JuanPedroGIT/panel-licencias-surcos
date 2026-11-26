# Surcos · Panel de licencias

Proyecto Vue 3 + Vite + TypeScript para Vercel. Conserva el componente original.

## Local
Node.js 22. Ejecutar `npm install`, `npm run dev` y `npm run build`.

## Vercel
Subir el contenido de esta carpeta a un repositorio Git. En Vercel, Add New > Project, importar el repositorio y seleccionar la carpeta raíz que contiene package.json. Framework Vite, Node.js 22, build `npm run build`, salida `dist`. Deploy. Alternativamente ejecutar `npx vercel` y después `npx vercel --prod` desde esta carpeta.

## Seguridad y Firebase
No necesita claves ni variables de entorno. El panel genera JSON localmente; no lee ni escribe Firebase. El estado publicado se introduce manualmente. Para publicar, copiar el JSON en Remote Config del proyecto surcos-dev, parámetro surcos_access_policy, y publicar desde una cuenta autorizada.
El despliegue es público por defecto. noindex no es autenticación. Para restringir acceso, activar Deployment Protection en Vercel y comprobar que cubre la URL de producción (según plan); si no está disponible, implementar autenticación real antes de considerar privado el panel. No introducir credenciales en el frontend.

## Validación
Ejecutar npm install y npm run build antes de publicar. No se ha verificado la compilación en el entorno de preparación por falta de dependencias instaladas.
