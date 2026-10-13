# 🦉 Taller: IA de Bolsillo: Automatiza tu terminal con Ollama y Modelos SLM

¡Bienvenidos, al IX Simposio de Linux en la Universidad Tecnológica de la Mixteca (UTM)!

Este repositorio contiene todo el material necesario para el taller práctico de integración de Modelos de Lenguaje Pequeños (SLM) en aplicaciones locales. Aprenderemos haciendo, construyendo un asistente de terminal desde cero.

## 🎯 1. Propósito del Taller

**Objetivo General:**
Demostrar mediante un ejercicio práctico y escalable cómo integrar un entorno de Inteligencia Artificial utilizando Ollama con la menor cantidad de recursos de cómputo posible (sobre Podman), para desarrollar una aplicación cliente en Rust que consuma estos modelos.

**Objetivos de Aprendizaje (Lo que lograrás hoy):**

* **Comprender** la diferencia entre LLMs (Large Language Models) y SLMs (Small Language Models) y el valor de la inferencia local.
* **Desplegar** un entorno de IA aislado y ligero utilizando contenedores sin demonio (*daemonless*) con Podman.
* **Desarrollar** un cliente HTTP asíncrono en Rust para interactuar con la API de Ollama.
* **Optimizar** el consumo de recursos (CPU/RAM) en entornos de hardware limitados.

## 🛠️ 2. Requerimientos y Preparación del Entorno (Software)

Antes de iniciar con los módulos, prepararemos nuestra estación de trabajo en **Ubuntu 24.04 LTS**. Abre tu terminal y ejecuta los siguientes bloques paso a paso.

### A. Instalación de Dependencias Base y Podman
Instalaremos los compiladores necesarios y nuestro gestor de contenedores.
```bash
sudo apt update
sudo apt install -y curl git build-essential podman
```

### B. Instalación de Rust y Cargo
Usaremos `rustup`, el instalador oficial de Rust.
```bash
# Descargar e instalar Rust automáticamente
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

# Cargar el entorno de Rust en la terminal actual
source "$HOME/.cargo/env"

# Verificar instalación
rustc --version
cargo --version
```

### C. Configuración de Vim para Desarrollo en Rust
Para programar directo en la terminal, usaremos `vim`. Le añadiremos el plugin oficial `rust.vim` para tener resaltado de sintaxis, aprovechando el sistema de paquetes nativo de Vim 8+.
```bash
# 1. Instalar Vim
sudo apt install -y vim

# 2. Crear la estructura de directorios para plugins nativos
mkdir -p ~/.vim/pack/plugins/start

# 3. Clonar el plugin oficial de Rust
git clone https://github.com/rust-lang/rust.vim ~/.vim/pack/plugins/start/rust.vim

# 4. Habilitar la sintaxis y los plugins en el archivo de configuración
echo "syntax enable" >> ~/.vimrc
echo "filetype plugin indent on" >> ~/.vimrc
```

## 🧑‍💻 3. Requerimientos Técnicos para los Asistentes

**Hardware Mínimo:**
* **Procesador:** 2 núcleos (Arquitectura x86_64).
* **Memoria RAM:** 4 GB de RAM (Se requieren al menos 2 GB libres durante la ejecución).
* **Almacenamiento:** 5 GB de espacio libre en disco.
* *No se requiere GPU dedicada.*

**Conocimientos Previos:**
* Navegación básica en la terminal de Linux (`cd`, `ls`, `mkdir`).
* Conceptos básicos de programación y peticiones HTTP.
* Uso básico de un editor de texto en terminal (Vim o Nano).

## 📚 4. Desarrollo del Contenido

### ⏱️ Módulo 1: Arquitectura Ligera e Inferencia Local (30 min)

**El paradigma del "Edge AI" (IA en el Borde)**
Históricamente, la IA generativa dependía de enormes centros de datos y APIs en la nube. Hoy, gracias a la cuantización, tenemos **SLMs (Small Language Models)**.

* **LLM vs SLM:** Mientras que un modelo como GPT-4 requiere clústeres de GPUs, modelos como Qwen 2.5 (0.5B) están diseñados para correr en la CPU de tu laptop.
* **¿Por qué Podman?:** Es *daemonless* (no hay proceso de fondo consumiendo RAM) y *rootless* (no requiere permisos root), perfecto para la universidad.
* **La Arquitectura de hoy:**
  `Tu App (Rust)` ↔️ `Petición HTTP (JSON)` ↔️ `Podman (Ollama)` ↔️ `Modelo SLM`

### 🐳 Módulo 2: Despliegue de Ollama con Podman (45 min)

Vamos a levantar nuestro "cerebro" local.

**Paso 1: Crear almacenamiento persistente**
Para evitar descargar el modelo si reiniciamos el contenedor:
```bash
podman volume create ollama_storage
```

**Paso 2: Levantar el servidor de Ollama**
```bash
podman run -d \
  --name ollama_slim \
  --net=host \
  -v ollama_storage:/root/.ollama \
  docker.io/ollama/ollama:latest
```
*Nota:* `--net=host` permite que nuestra app en Rust encuentre a Ollama en `localhost:11434`.

**Paso 3: Descargar y probar nuestro SLM**
```bash
podman exec -it ollama_slim ollama run qwen2.5:0.5b "Dime un chiste corto."
```

### 🦀 Módulo 3: Construcción del Asistente CLI en Rust (75 min)

**Paso 1: Inicializar el proyecto y abrir Vim**
```bash
cargo new term-assistant
cd term-assistant
vim Cargo.toml
```

**Paso 2: Configurar dependencias (`Cargo.toml`)**
Añade lo siguiente al final del archivo (`i` para insertar en Vim, `ESC` y `:wq` para guardar):
```toml
[dependencies]
tokio = { version = "1.38", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**Paso 3: Programar la lógica (`src/main.rs`)**
Abre `src/main.rs` con Vim: `vim src/main.rs`. Borra su contenido e inserta esto:

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::io::{self, Write};

#[derive(Serialize)]
struct OllamaRequest {
    model: String,
    prompt: String,
    stream: bool,
    system: String,
}

#[derive(Deserialize)]
struct OllamaResponse {
    response: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new();
    let api_url = "http://localhost:11434/api/generate";

    println!("🤖 === Asistente de Terminal UTM ===");
    print!("> ¿Qué comando de Ubuntu necesitas?: ");
    io::stdout().flush()?;

    let mut user_input = String::new();
    io::stdin().read_line(&mut user_input)?;

    let system_prompt = "Eres un experto en terminal Ubuntu Linux. \
        Responde SOLO con el comando exacto de Bash. \
        No agregues explicaciones ni formato Markdown.".to_string();

    let request_body = OllamaRequest {
        model: "qwen2.5:0.5b".to_string(),
        prompt: user_input.trim().to_string(),
        stream: false,
        system: system_prompt,
    };

    println!("Consultando a tu IA de bolsillo...");
    let res = client.post(api_url).json(&request_body).send().await?;

    if res.status().is_success() {
        let response_data: OllamaResponse = res.json().await?;
        println!("\n✅ [Comando]:\n{}", response_data.response.trim());
    } else {
        eprintln!("❌ Error: Status {}", res.status());
    }

    Ok(())
}
```

### 🧪 Módulo 4: Pruebas, Optimización y Cierre (30 min)

**1. Ejecutar y Probar**
```bash
cargo run --release
```

**2. Auditoría de Recursos**
En otra terminal, mientras el modelo responde, ejecuta:
```bash
podman stats ollama_slim
```
*Observa cómo corre un cerebro artificial consumiendo una fracción de RAM en comparación con otras herramientas pesadas.*

**3. Reto Challenge 🏆**
Modifiquen el código usando `std::process::Command` para que la app pregunte: `¿Deseas ejecutar este comando? [s/N]` y lo ejecute automáticamente en Ubuntu.
