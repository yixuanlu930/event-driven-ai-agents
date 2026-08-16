
# Distributed AI Task Processing System

A distributed, event-driven infrastructure for processing **text and image tasks using AI models**, built with Python, RabbitMQ, Flask, Docker, and Hugging Face.

The system combines **asynchronous message-based processing** with **synchronous REST APIs**, allowing AI agents to process automatically generated tasks while also accepting direct HTTP requests.

## Overview

This project implements a distributed task-processing architecture composed of multiple independent services.

A producer continuously generates two types of AI tasks:

* **Text sentiment analysis**
* **Image classification**

Tasks are distributed through **RabbitMQ** to specialized agents. Each agent processes its corresponding tasks, stores the results locally, and publishes processing information to a centralized logging service.

The agents also expose REST APIs, allowing tasks to be submitted and queried synchronously.

## Architecture

The system contains five main components:

### RabbitMQ

RabbitMQ acts as the messaging broker between the producer and the processing agents.

It enables asynchronous communication and decouples task generation from task processing.

### Task Producer

The producer automatically generates approximately one task per second.

Tasks can be:

* Text analysis tasks based on product review sentences
* Image classification tasks using images from the CIFAR-100 dataset

Each task contains:

```json
{
  "task_id": "unique_id",
  "type": "text_analysis or image_classification",
  "content": "task data",
  "routing_key": "routing information"
}
```

The producer publishes these tasks to RabbitMQ so they can be processed by the corresponding AI agent.

### Text Agent

The Text Agent performs **sentiment analysis** using the Hugging Face model:

```text
cardiffnlp/twitter-roberta-base-sentiment-latest
```

It can process tasks in two ways:

* Asynchronously through RabbitMQ
* Synchronously through its Flask REST API

For each task, the agent generates:

* Task ID
* Sentiment prediction
* Confidence score
* Timestamp

Results are stored in CSV files inside the `results/` directory.

### Image Agent

The Image Agent performs **image classification** using an EANet image-classification model obtained through Hugging Face.

Input images are generated from the **CIFAR-100 dataset**.

The agent supports both:

* Asynchronous task processing through RabbitMQ
* Synchronous processing through a REST API

The resulting class, confidence score, task identifier, and timestamp are persisted in CSV files.

### Global Logger

The Global Logger receives processing results from all agents using a RabbitMQ **fanout exchange**.

It creates a centralized log containing:

* Task ID
* Agent
* Result
* Confidence
* Timestamp

The global processing history is stored in:

```text
results/tasks_log.csv
```

## System Flow

```text
                    ┌─────────────────┐
                    │  Task Producer  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    RabbitMQ     │
                    └───────┬─────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
        ┌───────────────┐       ┌───────────────┐
        │  Text Agent   │       │  Image Agent  │
        │   Sentiment   │       │Classification │
        └───────┬───────┘       └───────┬───────┘
                │                       │
                └───────────┬───────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │ Global Logger   │
                    └─────────────────┘
```

## Features

* Distributed task processing
* Event-driven architecture
* RabbitMQ message broker
* AI-powered text sentiment analysis
* AI-powered image classification
* CIFAR-100 image generation
* Hugging Face model integration
* REST APIs with Flask
* Asynchronous and synchronous processing
* Persistent CSV results
* Centralized task logging
* Docker containerization
* Docker Compose orchestration
* Support for horizontal agent scaling

## Technologies

* **Python**
* **RabbitMQ**
* **Docker**
* **Docker Compose**
* **Flask**
* **Hugging Face**
* **TensorFlow**
* **PyTorch / Torchvision**
* **Pika**
* **NumPy**
* **CIFAR-100**
* **REST APIs**
* **CSV persistence**

## Project Structure

```text
distributed-ai-task-processing/
│
├── producer/
│   ├── event_generator.py
│   ├── producer.py
│   ├── publisher.py
│   ├── Dockerfile.producer
│   └── requirements.txt
│
├── agents/
│   ├── text_agent.py
│   ├── image_agent.py
│   ├── Dockerfile.text
│   ├── Dockerfile.image
│   └── requirements.txt
│
├── logger/
│   ├── global_logger.py
│   └── Dockerfile.logger
│
├── results/
│   └── generated CSV results
│
├── docker-compose.yml
├── LICENSE
└── README.md
```

## Deployment

The complete infrastructure is orchestrated using Docker Compose.

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/distributed-ai-task-processing.git
cd distributed-ai-task-processing
```

### 2. Configure Hugging Face

A Hugging Face access token is required by the AI agents.

For security reasons, credentials should not be committed directly to the repository.

Create a local `.env` file:

```text
HF_TOKEN=your_huggingface_token
```

Then configure Docker Compose to read the environment variable.

> Never commit personal API tokens or credentials to a public GitHub repository.

### 3. Start the infrastructure

```bash
docker compose up --build
```

To run it in the background:

```bash
docker compose up --build -d
```

Check the running containers:

```bash
docker compose ps
```

Stop the infrastructure with:

```bash
docker compose down
```

## RabbitMQ Management Interface

RabbitMQ provides a management interface at:

```text
http://localhost:15672
```

The interface can be used to inspect exchanges, queues, connections, and message activity.

## REST APIs

The two AI agents expose synchronous REST interfaces.

### Text Agent

```text
http://localhost:5001
```

### Image Agent

```text
http://localhost:5002
```

Both agents implement:

```text
POST /tasks
GET /tasks
GET /tasks/<task_id>
```

### Example: Submit a text task

```bash
curl -X POST http://localhost:5001/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "task_id": "demo_text_1",
    "content": "This product is amazing"
  }'
```

### List processed tasks

```bash
curl http://localhost:5001/tasks
```

### Retrieve a specific task

```bash
curl http://localhost:5001/tasks/demo_text_1
```

## Monitoring

Follow the producer logs:

```bash
docker compose logs -f producer
```

Text Agent:

```bash
docker compose logs -f text_agent
```

Image Agent:

```bash
docker compose logs -f image_agent
```

Global Logger:

```bash
docker compose logs -f global_logger
```

## Results

Processing results are stored in the `results/` directory.

Individual agents generate files such as:

```text
text_results_<container_id>.csv
image_results_<container_id>.csv
```

The Global Logger generates:

```text
tasks_log.csv
```

Using the container hostname in result filenames allows different agent instances to maintain separate output files when the architecture is scaled horizontally.

## Apple Silicon

On Apple Silicon systems such as M1, M2, or M3 Macs, some TensorFlow dependencies may require an `amd64` container.

If the Image Agent produces architecture-related errors, enable the following option for the `image_agent` service in `docker-compose.yml`:

```yaml
platform: linux/amd64
```

## Architectural Concepts

This project demonstrates several distributed-system concepts:

* Producer-consumer architecture
* Message brokers
* Event-driven communication
* Message queues
* Fanout exchanges
* Loose coupling
* Asynchronous processing
* REST-based synchronous communication
* Independent processing agents
* Containerized services
* Centralized logging
* Horizontal scalability

## Educational Purpose

This project was developed as part of a **Big Data Infrastructure** academic project.

Its primary purpose is to explore the design and implementation of distributed infrastructures combining messaging systems, containerized services, REST interfaces, and AI-based task processing.

## License

This project is distributed under the license included in the repository.




## Spanish transalation:

PRÁCTICA 1 - INFRAESTRUCTURAS DE BIG DATA
=========================================

1. Cómo desplegar la infraestructura del sistema
------------------------------------------------
El sistema se despliega con Docker Compose y levanta los siguientes servicios:

- rabbitmq: broker de mensajería
- producer: generador automático de tareas
- text_agent: agente de análisis de sentimientos
- image_agent: agente de clasificación de imágenes
- global_logger: logger global de tareas procesadas

Levantar la infraestructura:
  docker compose up --build

Ejecutarla en segundo plano:
  docker compose up --build -d

Comprobar que los contenedores están activos:
  docker compose ps

Detener la infraestructura:
  docker compose down

Importante sobre Hugging Face:
Para ejecutar correctamente el servicio text_agent es necesario disponer de un token personal de Hugging Face.

Aunque no sea la opción más segura, por simpleza para ejecutar el código se incluye directamente el token en el archivo docker-compose.yml. La opción más limpia es que cada usuario utilice su propio User Access Token de Hugging Face y lo configure en un archivo .env local.

Pasos:
1. Crear una cuenta en https://huggingface.co
2. Ir a https://huggingface.co/settings/tokens
3. Crear un nuevo token con:
   - Type: Read
   - Opción "Make calls to Inference Providers" activada
4. En el archivo docker-compose.yml, sustituir el valor de HF_TOKEN en el servicio text_agent y en image_agent por tu token personal:

  - HF_TOKEN=hf_tu_token_aqui

Sin este token, o con un token sin los permisos correctos, los agentes fallarán con un error 403 Forbidden al intentar conectarse a la Inference API de Hugging Face.

¡Importante sobre procesadores Apple Silicon (M1/M2/M3) u otros errores de arquitectura!:
Si al intentar levantar la infraestructura (al hacer el build) te aparece un error de compatibilidad o un "exec format error" en el contenedor image_agent, es muy probable que estés usando un Mac con procesador ARM (M1/M2/M3). Las librerías de TensorFlow pueden dar problemas al construirse en estas arquitecturas.

Para solucionarlo, debes abrir el archivo docker-compose.yml, buscar el servicio image_agent y descomentar (quitar la almohadilla #) la línea de la plataforma:

  #platform: linux/amd64

Dejándola así:
  platform: linux/amd64

Esto forzará a Docker a "usar" la arquitectura x86_64 para ese contenedor y solucionará el error de compilación.
--------------------------------------------------------------------------------

1. Cómo ejecutar los agentes
----------------------------
Los agentes se ejecutan como servicios definidos en docker-compose.yml, por lo que no es necesario lanzarlos manualmente fuera de Docker Compose.

Al arrancar la infraestructura con:
  docker compose up --build

se inician automáticamente:
- text_agent
- image_agent
- global_logger

Reiniciar un agente concreto:
  docker compose restart text_agent
  docker compose restart image_agent
  docker compose restart global_logger

Ver los logs de cada servicio:
  docker compose logs -f text_agent
  docker compose logs -f image_agent
  docker compose logs -f global_logger

--------------------------------------------------------------------------------

3. Cómo probar el funcionamiento del sistema
--------------------------------------------
Una vez desplegado el sistema, se puede comprobar su funcionamiento de varias formas.

Verificar que RabbitMQ está funcionando:
RabbitMQ dispone de interfaz web en:
  http://localhost:15672

Credenciales:
- usuario: user
- contraseña: password

Desde ahí se puede comprobar que las colas existen y que los mensajes se publican y consumen correctamente.

Verificar que el producer genera tareas:
  docker compose logs -f producer

Deberían aparecer mensajes indicando el envío continuo de tareas.

Verificar que los agentes procesan tareas:
  docker compose logs -f text_agent
  docker compose logs -f image_agent

Deberían verse mensajes de recepción y procesamiento.

Verificar que se generan resultados:
Los resultados se almacenan en la carpeta results/.

Para comprobarlo:
  ls results

Y para ver el contenido de un archivo CSV:
  head results/nombre_del_fichero.csv

Los agentes generan archivos CSV locales, por ejemplo:
- text_results_<hostname>.csv
- image_results_<hostname>.csv

Además, el servicio global_logger genera un archivo global:
- tasks_log.csv

Verificar la API síncrona de los agentes:
- text_agent expone su API en: http://localhost:5001
- image_agent expone su API en: http://localhost:5002

Operaciones disponibles:
- POST /tasks -> enviar una nueva tarea al agente
- GET /tasks -> consultar las tareas conocidas por el agente
- GET /tasks/<task_id> -> consultar una tarea concreta

Ejemplo para probar text_agent:
  curl -X POST http://localhost:5001/tasks \
    -H "Content-Type: application/json" \
    -d '{"task_id":"demo_text_1","content":"This product is amazing"}'

Ejemplo para consultar tareas del text_agent:
  curl http://localhost:5001/tasks

Ejemplo para consultar una tarea concreta:
  curl http://localhost:5001/tasks/demo_text_1

--------------------------------------------------------------------------------

4. Breve descripción de la arquitectura implementada
----------------------------------------------------
La arquitectura implementada sigue un modelo distribuido orientado a eventos.

El producer genera tareas automáticamente y las publica en RabbitMQ. RabbitMQ actúa como intermediario entre la generación de tareas y su procesamiento, permitiendo desacoplar ambos componentes.

Los agentes consumen las tareas desde sus colas correspondientes:
- text_agent procesa tareas de análisis de sentimientos
- image_agent procesa tareas de clasificación de imágenes

Cada agente mantiene el procesamiento asíncrono original mediante RabbitMQ y, además, expone una API HTTP REST para interacción síncrona. De este modo, cada agente puede:
- consumir tareas de forma asíncrona
- aceptar peticiones síncronas vía HTTP
- devolver información en formato JSON sobre las tareas procesadas

Cada agente guarda sus resultados en archivos CSV dentro de la carpeta results/. Para permitir escalabilidad horizontal, cada instancia puede escribir en un fichero distinto usando el hostname del contenedor como parte del nombre del archivo.

Además, se ha añadido un servicio global_logger que recibe los resultados procesados por los agentes a través de RabbitMQ y los registra en un único archivo global (tasks_log.csv) con información de:
- task_id
- agent
- result
- confidence
- timestamp

Esta arquitectura permite:
- procesamiento asíncrono
- interacción síncrona mediante API REST
- separación entre productor y consumidores
- persistencia de resultados locales y globales
- posibilidad de escalar los agentes en paralelo
