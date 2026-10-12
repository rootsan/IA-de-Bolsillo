# 🦉 Taller: IA de Bolsillo: Automatiza tu terminal con Ollama y Modelos SLM

¡Bienvenidos, al XI de la Universidad Tecnológica de la Mixteca (UTM)! 

Este repositorio contiene todo el material necesario para el taller práctico de integración de Modelos de Lenguaje Pequeños (SLM) en aplicaciones locales. Aprenderemos haciendo, construyendo un asistente de terminal desde cero.

---

## 🎯 1. Propósito del Taller

**Objetivo General:**
Demostrar mediante un ejercicio práctico y escalable cómo integrar un entorno de Inteligencia Artificial utilizando Ollama con la menor cantidad de recursos de cómputo posible (sobre Podman), para desarrollar una aplicación cliente en Rust que consuma estos modelos.

**Objetivos de Aprendizaje (Lo que lograrás hoy):**
*   **Comprender** la diferencia entre LLMs (Large Language Models) y SLMs (Small Language Models) y el valor de la inferencia local.
*   **Desplegar** un entorno de IA aislado y ligero utilizando contenedores sin demonio (*daemonless*) con Podman.
*   **Desarrollar** un cliente HTTP asíncrono en Rust para interactuar con la API de Ollama.
*   **Optimizar** el consumo de recursos (CPU/RAM) en entornos de hardware limitados.

---

## 🛠️ 2. Requerimientos Técnicos para el Despliegue (Software)

Para que el taller funcione de manera óptima, el entorno de desarrollo debe contar con:

1.  **Sistema Operativo:** Ubuntu 24.04 LTS (Nativo o en Máquina Virtual).
2.  **Gestor de Contenedores:** `podman` (versión 4.x o superior).
3.  **Lenguaje de Programación:** `rustc` y su gestor de paquetes `cargo` (instalados vía `rustup`).
4.  **Imagen de Contenedor:** `docker.io/ollama/ollama:latest`.
5.  **Modelo SLM (Small Language Model):** `qwen2.5:0.5b` (Parámetros: 0.5 Billones, Tamaño en disco: ~350 MB, Consumo RAM: ~400 MB).

> 💡 **Nota para el Facilitador:** Se recomienda tener un *mirror* local o memorias USB con las imágenes de Podman y el caché de Cargo (crates) para evitar la saturación de la red de la universidad durante el taller.

---

## 🧑‍💻 3. Requerimientos Técnicos para los Asistentes

**Hardware Mínimo:**
*   **Procesador:** 2 núcleos (Arquitectura x86_64).
*   **Memoria RAM:** 4 GB de RAM (Se requieren al menos 2 GB libres durante la ejecución).
*   **Almacenamiento:** 5 GB de espacio libre en disco.
*   *No se requiere GPU dedicada.*

**Conocimientos Previos (Deseables pero no obligatorios):**
*   Navegación básica en la terminal de Linux (`cd`, `ls`, `mkdir`).
*   Conceptos básicos de programación estructurada/orientada a objetos.
*   Familiaridad básica con formatos de datos como JSON.

---

## 📚 4. Desarrollo del Contenido

### ⏱️ Módulo 1: Arquitectura Ligera e Inferencia Local (30 min)

**El paradigma del "Edge AI" (IA en el Borde)**
Históricamente, la IA generativa dependía de enormes centros de datos y APIs en la nube (como OpenAI o Anthropic). Hoy, gracias a la cuantización y la optimización de arquitecturas, tenemos **SLMs (Small Language Models)**.

*   **LLM vs SLM:** Mientras que un modelo como GPT-4 tiene billones de parámetros y requiere clústeres de GPUs, modelos como Qwen 2.5 (0.5B) o Llama 3.2 (1B) están diseñados para correr en la CPU de una laptop estándar.
*   **¿Por qué Podman y no Docker?:** En la UTM promovemos las mejores prácticas de seguridad y eficiencia. Podman es *daemonless* (no hay un proceso corriendo siempre de fondo consumiendo RAM) y *rootless* (no requiere permisos de superusuario), lo que lo hace perfecto para nuestro entorno académico.
*   **La Arquitectura de hoy:**
    `Tu App en Rust` ↔️ `Petición HTTP (JSON)` ↔️ `Contenedor Podman (Ollama)` ↔️ `Modelo SLM`

---

### 🐳 Módulo 2: Despliegue de Ollama con Podman & SLMs (45 min)

¡Manos a la obra! Vamos a levantar nuestro "cerebro" local.

**Paso 1: Instalar Podman**
```bash
sudo apt update && sudo apt install -y podman
```

**Paso 2: Crear almacenamiento persistente**
Para evitar descargar el modelo de IA cada vez que reiniciemos el contenedor, creamos un volumen:
```bash
podman volume create ollama_storage
```

**Paso 3: Levantar el servidor de Ollama**
```bash
podman run -d \
  --name ollama_slim \
  --net=host \
  -v ollama_storage:/root/.ollama \
  docker.io/ollama/ollama:latest
```
*💡 Pedagógicamente:* Nota cómo usamos `--net=host`. Esto permite que nuestra app en Rust encuentre a Ollama directamente en `localhost:11434`.

**Paso 4: Descargar y probar nuestro SLM**
```bash
podman exec -it ollama_slim ollama run qwen2.5:0.5b "Dime un chiste corto de programadores."
```
¡Si el modelo te responde, tu servidor de IA está listo!

---

### 🦀 Módulo 3: Construcción del Asistente CLI en Rust (75 min)

Ahora construiremos un cliente súper rápido y seguro en memoria usando **Rust**. Nuestra app tomará lenguaje natural y devolverá un comando de Linux.

**Paso 1: Inicializar el proyecto**
```bash
cargo new term-assistant
cd term-assistant
```

**Paso 2: Configurar dependencias (`Cargo.toml`)**
Abre el archivo `Cargo.toml` y añade las librerías necesarias. Usaremos `tokio` para asincronismo y `reqwest` para peticiones HTTP.
```toml
[package]
name = "term-assistant"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.38", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**Paso 3: Programar la lógica (`src/main.rs`)**
Reemplaza el contenido de `src/main.rs` con el siguiente código. Lee los comentarios, ahí está la magia de cómo estructuramos el *System Prompt* para que la IA actúe como una máquina y no como un conversador.

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::io::{self, Write};

// 1. Definimos la estructura de la petición que espera Ollama
#[derive(Serialize)]
struct OllamaRequest {
    model: String,
    prompt: String,
    stream: bool,
    system: String, // Aquí le daremos la "personalidad"
}

// 2. Definimos cómo leer la respuesta
#[derive(Deserialize)]
struct OllamaResponse {
    response: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new();
    let api_url = "http://localhost:11434/api/generate";

    println!("🤖 === Asistente de Terminal UTM (Rust + SLM) ===");
    print!("> ¿Qué comando de Ubuntu necesitas hoy?: ");
    io::stdout().flush()?;

    let mut user_input = String::new();
    io::stdin().read_line(&mut user_input)?;

    // 3. El "System Prompt" es vital. Es la instrucción maestra para el modelo.
    let system_prompt = "Eres un asistente experto en la terminal de Ubuntu Linux. \
        Tu única tarea es responder con el comando exacto de Bash para la petición del usuario. \
        No agregues explicaciones, no uses formato Markdown, ni saludos. Devuelve SOLO el comando."
        .to_string();

    let request_body = OllamaRequest {
        model: "qwen2.5:0.5b".to_string(),
        prompt: user_input.trim().to_string(),
        stream: false,
        system: system_prompt,
    };

    println!("Consultando a tu IA de bolsillo...");

    // 4. Hacemos la petición asíncrona
    let res = client.post(api_url).json(&request_body).send().await?;

    if res.status().is_success() {
        let response_data: OllamaResponse = res.json().await?;
        println!("\n✅ [Comando Sugerido]:");
        println!("{}\n", response_data.response.trim());
    } else {
        eprintln!("❌ Error al comunicarse con Ollama: Status {}", res.status());
    }

    Ok(())
}
```

---

### 🧪 Módulo 4: Pruebas, Optimización y Cierre (30 min)

**1. Ejecutar y Probar**
Compilamos nuestro programa en modo *release* (para máxima velocidad y optimización):
```bash
cargo run --release
```
*Prueba con:* "Busca todos los archivos .txt en mi carpeta de descargas que pesen más de 10MB".

**2. Auditoría de Recursos (El momento de la verdad)**
Mientras el modelo está generando una respuesta, abre otra terminal y ejecuta:
```bash
podman stats ollama_slim
```
*Reflexión para los alumnos:* Observen el uso de la memoria RAM y la CPU. ¡Acaban de correr un cerebro artificial con menos recursos de los que consume una pestaña de Google Chrome!

**3. Reto final para casa (Challenge) 🏆**
Como proyecto para mejorar sus habilidades, modifiquen el código en Rust usando la librería nativa `std::process::Command` para que la aplicación:
1. Muestre el comando generado.
2. Pregunte al usuario: `¿Deseas ejecutar este comando? [s/N]`.
3. Si el usuario presiona `s`, que el programa en Rust ejecute el comando en la terminal automáticamente.

---

*¡Gracias por participar! Sigamos construyendo tecnología eficiente y accesible.*
