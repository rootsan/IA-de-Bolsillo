# 🐳 Mini How-To: Uso Básico de Podman

Podman es un motor de contenedores compatible con la sintaxis de Docker, pero con una ventaja crucial: **no requiere permisos de administrador (rootless) ni ejecuta un demonio en segundo plano (daemonless)**, lo que ahorra RAM y mejora la seguridad.

Si sabes usar Docker, ya sabes usar Podman. A continuación, los comandos más útiles para el taller:

## 1. Gestión de Imágenes

**Buscar una imagen en el registro:**
```bash
podman search ubuntu
```

**Descargar una imagen:**
```bash
podman pull docker.io/ollama/ollama:latest
```

**Listar las imágenes descargadas en tu disco:**
```bash
podman images
```

## 2. Ejecución y Gestión de Contenedores

**Crear y ejecutar un contenedor en segundo plano (`-d`):**
```bash
podman run -d --name mi_servidor -p 8080:80 docker.io/nginx
```
*(Nota: `-p 8080:80` mapea el puerto 8080 de tu PC al puerto 80 del contenedor).*

**Ver contenedores en ejecución:**
```bash
podman ps
```

**Ver TODOS los contenedores (incluyendo los apagados):**
```bash
podman ps -a
```

## 3. Interactuar con un Contenedor

**Ver cuánto CPU y RAM está usando un contenedor en tiempo real:**
```bash
podman stats <nombre_del_contenedor>
```

**Ver los logs (útil para detectar errores):**
```bash
podman logs -f <nombre_del_contenedor>
```

**Ejecutar un comando dentro del contenedor activo:**
```bash
# Entrar a la terminal del contenedor:
podman exec -it <nombre_del_contenedor> /bin/bash

# Ejecutar un comando rápido y salir (ej. lo que usamos en el taller):
podman exec -it ollama_slim ollama run qwen2.5:0.5b "Hola"
```

## 4. Limpieza (Detener y Borrar)

**Detener un contenedor en ejecución:**
```bash
podman stop <nombre_del_contenedor>
```

**Iniciar un contenedor detenido:**
```bash
podman start <nombre_del_contenedor>
```

**Borrar un contenedor (debe estar detenido):**
```bash
podman rm <nombre_del_contenedor>
```

**Borrar una imagen para liberar espacio:**
```bash
podman rmi <id_o_nombre_de_imagen>
```
