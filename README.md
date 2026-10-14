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

## 🧑‍💻 2. Requerimientos Técnicos

**Hardware Mínimo:**
* **Procesador:** 2 núcleos (Arquitectura x86_64).
* **Memoria RAM:** 4 GB de RAM (Se requieren al menos 2 GB libres durante la ejecución).
* **Almacenamiento:** 10 GB de espacio libre en disco.
* *No se requiere GPU dedicada.*

**Conocimientos Previos:**
* Navegación básica en la terminal de Linux (`cd`, `ls`, `mkdir`, redirecciones y pipes).
* Conceptos básicos de programación estructurada manejo de variables, control de flujo, funciones.
* Uso básico de un editor de texto en terminal (Vim).

## 🛠️ 3. Requerimientos y Preparación del Entorno (Software)

Antes de iniciar con los módulos, prepararemos nuestra estación de trabajo en **Ubuntu 24.04 LTS**. Abre tu terminal y ejecuta los siguientes bloques paso a paso.

### A. Instalación de Dependencias Base y Podman
Instalaremos los compiladores necesarios y nuestro gestor de contenedores.
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git build-essential podman rustc cargo vim curl
```

### C. Configuración de Vim para Desarrollo en Rust
Para programar directo en la terminal, usaremos `vim`. Crea la configuración de Vim para habilitar resaltado de sintaxis, sangría inteligente y numeración de líneas:
```bash
cat << 'EOF' > ~/.vimrc
syntax on
filetype plugin indent on
set number
set tabstop=4
set shiftwidth=4
set expandtab
set autoindent
colorscheme desert
EOF
```
## 📚 4. Desarrollo del Contenido

### ⏱️ Módulo 1: Arquitectura Ligera e Inferencia Local (30 min)

**El paradigma del "Edge AI" (IA en el Borde)**
Históricamente, la IA generativa dependía de enormes centros de datos y APIs en la nube. Hoy, gracias a la cuantización, tenemos **SLMs (Small Language Models)**.

* **LLM vs SLM:** Mientras que un modelo como GPT-4 requiere clústeres de GPUs, modelos como Qwen 2.5 (0.5B) están diseñados para correr en la CPU de tu laptop. Razones por las cuales un modelo de 500M de parámetros (qwen2.5:0.5b) es óptimo para tareas deterministas (traducción de lenguaje natural a comandos Bash), manteniendo un consumo de memoria inferior a 500 MB.
* **¿Por qué Podman?:** Es *daemonless* (no hay proceso de fondo consumiendo RAM) y *rootless* (no requiere permisos root), perfecto para la universidad y para entornos productivos disminuyendo el nivel de exposición.
* **La Arquitectura de hoy:**
Comprensión del endpoint REST /api/generate y el formato de intercambio JSON sobre HTTP.

### 🐳 Módulo 2: Despliegue de Ollama con Podman (45 min)

Vamos a levantar nuestro "cerebro" local.

**Paso 1: Crear almacenamiento persistente**
Garantiza que los modelos descargados persistan entre reinicios del contenedor:
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
podman exec -it ollama_slim ollama run qwen2.5:0.5b "Dime un chiste corto de programadores."
```
***Paso 4: Validar la API local con curl:**
```bash
curl -s [http://127.0.0.1:11434/api/generate](http://127.0.0.1:11434/api/generate) -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "ping al localhost",
  "stream": false,
  "system": "Devuelve solo el comando bash exacto"
}'
```
### 🦀 Módulo 3: Construcción del Asistente CLI en Rust (Arquitectura TCP Stream) (75 min)
Para evitar incompatibilidades de versiones con crates.io, clientes HTTP pesados o runtimes asíncronos que incrementan el tiempo de compilación, el asistente se comunica con Ollama utilizando sockets TCP directos (std::net::TcpStream) provistos por la biblioteca estándar de Rust, dejando únicamente serde para la manipulación de JSON.

**Paso 1: Inicializar el proyecto y abrir Vim**
```bash
cargo new term-assistant
cd term-assistant
vim Cargo.toml
```

**Paso 2: Configurar dependencias (`Cargo.toml`)**
Añade lo siguiente al final del archivo (`i` para insertar en Vim, `ESC` y `:wq` para guardar):
```toml
[package]
name = "asistente_cli"
version = "0.1.0"
edition = "2021"
##Añade solo las apartir de aquí
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**Paso 3: Programar la lógica (`src/main.rs`)**
Implementación del cliente nativo. Abre `src/main.rs` con Vim: `vim src/main.rs`. Inserta el siguiente código:

```rust
use serde::{Deserialize, Serialize};
use std::env;
use std::io::{self, Read, Write};
use std::net::TcpStream;

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

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Captura de argumentos CLI o lectura interactiva desde stdin
    let args: Vec<String> = env::args().skip(1).collect();

    let user_prompt = if !args.is_empty() {
        args.join(" ")
    } else {
        println!("=== IA de Bolsillo: Asistente CLI ===");
        print!("¿Qué comando necesitas?: ");
        io::stdout().flush()?;
        let mut buffer = String::new();
        io::stdin().read_line(&mut buffer)?;
        buffer.trim().to_string()
    };

    if user_prompt.trim().is_empty() {
        eprintln!("Error: Instrucción vacía.");
        return Ok(());
    }

    // 2. Definición de directiva del sistema
    let system_instructions = "Eres un asistente experto en Linux Bash. \
        Tu única tarea es devolver el comando exacto que realiza la acción solicitada. \
        No agregues introducciones, explicaciones, markdown, bloques de código ni comillas. \
        Devuelve estrictamente el comando en texto plano."
        .to_string();

    let request_body = OllamaRequest {
        model: "qwen2.5:0.5b".to_string(),
        prompt: user_prompt,
        stream: false,
        system: system_instructions,
    };

    let payload = serde_json::to_string(&request_body)?;

    // 3. Conexión directa mediante socket TCP a Ollama (localhost:11434)
    let mut stream = TcpStream::connect("127.0.0.1:11434")?;

    // 4. Construcción y envío de petición HTTP/1.0 manual
    let http_request = format!(
        "POST /api/generate HTTP/1.0\r\n\
         Host: 127.0.0.1:11434\r\n\
         Content-Type: application/json\r\n\
         Content-Length: {}\r\n\
         Connection: close\r\n\r\n\
         {}",
        payload.len(),
        payload
    );

    stream.write_all(http_request.as_bytes())?;

    // 5. Lectura de la respuesta del servidor
    let mut raw_response = String::new();
    stream.read_to_string(&mut raw_response)?;

    // 6. Extracción y parseo del payload JSON
    if let (Some(start), Some(end)) = (raw_response.find('{'), raw_response.rfind('}')) {
        let json_body = &raw_response[start..=end];
        let data: OllamaResponse = serde_json::from_str(json_body)?;
        println!("\n[Comando Sugerido]:\n{}", data.response.trim());
    } else {
        eprintln!("Error: Formato de respuesta no reconocido.");
    }

    Ok(())
}
```

### 🧪 Módulo 4: Compilación, Despliegue Global y Auditoría (30 min)

**1. Ejecutar y Probar**
Pasa el comando directamente entre comillas como argumento del binario:
```bash
cargo run --release -- "comprimir la carpeta actual en un tar gz excluyendo archivos log"
```
**2. Compilar el binario standalone optimizado**
```bash
cargo build --release
```
El archivo ejecutable resultante se genera en target/release/asistente_cli.

**3. Pruebalo**
```bash
./target/release/asistente_cli "mostrar los 5 procesos que consumen mas memoria"
```
**4. Auditoría de Recursos**

En otra terminal, mientras el modelo responde, ejecuta:
```bash
podman stats ollama_slim
```
*Observa cómo corre un cerebro artificial consumiendo una fracción de RAM en comparación con otras herramientas pesadas.*

**5. Instalación como comando global del sistema**

Opción A: Instalación a nivel de usuario (Recomendada, sin requerir sudo). Ubuntu 24.04 incluye ~/.local/bin en la variable $PATH de la sesión del usuario por defecto:
```bash
mkdir -p ~/.local/bin
cp target/release/asistente_cli ~/.local/bin/
```
Opción B: Instalación global para todos los usuarios del sistema
```bash
sudo cp target/release/asistente_cli /usr/local/bin/
```
**6. Validación de ejecución global 🏆**

Navega a cualquier directorio y llama a tu asistente como cualquier comando estándar de Linux:
```bash
cd ~
asistente_cli "buscar archivos modificados en los ultimos 3 dias"
```
### 🛡️ Alternativa de Contingencia: Modo Cero Dependencias (Sin Internet)
Si durante el taller la conexión a Internet en el laboratorio experimenta caídas totales y Cargo no puede descargar paquetes desde crates.io, los alumnos pueden implementar la herramienta utilizando exclusivamente la biblioteca estándar de Rust y delegando la petición HTTP a la utilidad curl del sistema operativo mediante std::process::Command.
**1. Cargo.toml sin librerías externas**
```txt
[package]
name = "asistente_cli"
version = "0.1.0"
edition = "2021"

[dependencies]
# Cero dependencias externas requeridas
```
**2. Implementación de contingencia en src/main.rs**
Edita src/main.rs:
```rust
use std::env;
use std::io::{self, Write};
use std::process::Command;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args: Vec<String> = env::args().skip(1).collect();

    let user_prompt = if !args.is_empty() {
        args.join(" ")
    } else {
        print!("¿Qué comando necesitas?: ");
        io::stdout().flush()?;
        let mut buffer = String::new();
        io::stdin().read_line(&mut buffer)?;
        buffer.trim().to_string()
    };

    if user_prompt.trim().is_empty() {
        eprintln!("Error: Instrucción vacía.");
        return Ok(());
    }

    let payload = format!(
        r#"{{"model": "qwen2.5:0.5b", "prompt": "{}", "stream": false, "system": "Responde estrictamente con el comando bash, sin markdown ni explicaciones."}}"#,
        user_prompt.replace('"', "\\\"")
    );

    let output = Command::new("curl")
        .args([
            "-s",
            "-X", "POST",
            "[http://127.0.0.1:11434/api/generate](http://127.0.0.1:11434/api/generate)",
            "-H", "Content-Type: application/json",
            "-d", &payload,
        ])
        .output()?;

    let response_text = String::from_utf8_lossy(&output.stdout);

    // Extracción manual del campo "response" sin analizador JSON externo
    if let Some(pos) = response_text.find("\"response\":\"") {
        let remainder = &response_text[pos + 12..];
        if let Some(end) = remainder.find("\",\"") {
            let cmd = &remainder[..end];
            println!("\n[Comando Sugerido]:\n{}", cmd.replace("\\n", "\n").replace("\\\"", "\""));
            return Ok(());
        }
    }

    println!("\n[Respuesta Bruta]:\n{}", response_text);
    Ok(())
}
```
**3. Compilación e instalación inmediata**
Esta versión compila en menos de 2 segundos sin conectarse a internet:
```
cargo build --release
cp target/release/asistente_cli ~/.local/bin/
```
