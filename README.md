# 🚀 AWS Engineering Repository

**Author:** Edgar Macías — Senior Software Engineer & AppSec Advocate

Este repositorio documenta y consolida mi proceso profesional de adopción, dominio y aplicación de **Amazon Web Services (AWS)** desde la perspectiva de un **ingeniero de software sénior**, con enfoque en:

* Arquitectura moderna
* Seguridad desde el diseño (Security by Design)
* Infraestructura como código (Terraform)
* Buenas prácticas de ingeniería en la nube
* Desarrollo de servicios serverless (Go + AWS Lambda)
* Gobernanza, control de costos y operación segura

El objetivo no es ser un “roadmap de aprendizaje”, sino construir un **cuerpo de trabajo técnico verificable**, con estándares profesionales y material que refleje la forma en que implemento, documento y despliego soluciones en AWS.

---

## 📘 **Propósito del Repositorio**

Este proyecto funciona como un **laboratorio estructurado**, donde desarrollo:

* Componentes reales listos para producción
* Arquitecturas modulares basadas en principios hexagonales
* Configuración segura y reproducible de infraestructura
* Adaptadores, servicios y utilidades para entornos cloud
* Lineamientos de seguridad aplicables a repositorios públicos

Cada módulo representa aprendizajes prácticos y decisiones de diseño que un **AWS Engineer, Cloud Developer o Backend Engineer** enfrentaría en proyectos reales.

---

## 🧱 **Arquitectura y Estructura del Proyecto**

La estructura del proyecto sigue arquitectura hexagonal (Ports & Adapters), diseñada para permitir extensibilidad, pruebas limpias y compatibilidad con AWS Lambda y otros servicios.

Estructura base:


Incluye:

### **`internal/core`**

Dominios, servicios y puertos. Aquí vive la lógica de negocio desacoplada de AWS u otros proveedores.

### **`internal/adapters`**

Adaptadores para almacenamiento, autenticación, rate limiting, bases de datos, DynamoDB, RDS, Redis, etc.

### **`cmd/api`**

Entrypoint principal para la API o funciones serverless empaquetadas para Lambda.

### **`internal/config`**

Gestión de configuración, env vars y parámetros remotos.

### **`Dockerfile` & `docker-compose.yml`**

Ambiente de desarrollo portable, reproducible y seguro.

---

## 🎓 **Plan de Estudios Técnico (Blueprint Profesional)**

Este repositorio contiene un **syllabus estructurado**, orientado a dominar AWS con enfoque profesional y aplicable a entornos reales:



Este plan está diseñado para crear una base sólida como AWS Engineer:

* Gobernanza y seguridad desde el inicio
* Terraform seguro y backend remoto
* IAM con mínimo privilegio
* Serverless en Go para producción
* Adaptadores para DynamoDB y RDS
* Arquitectura limpia compatible con Lambda

Cada módulo tendrá su propio subdirectorio, ejemplos prácticos, documentación y componentes reutilizables.

---

## 🔐 **Reglas de Seguridad para un Repositorio Público**

Este repositorio está diseñado para mantenerse completamente seguro como entorno de práctica y despliegue:



**Garantizamos que:**

* No se almacenan secretos, credenciales ni `.tfstate`.
* Todos los secretos deben pasar mediante variables `TF_VAR_`.
* Se evita el uso de Access Keys persistentes.
* Se aplican flujos de pre-commit y verificación manual.

Esto convierte al repositorio en un **ejemplo seguro** de prácticas para desarrolladores que trabajan con AWS.

---

## 🛠️ **Tecnologías Clave**

* **AWS:** IAM, Lambda, DynamoDB, API Gateway, Budgets, S3, RDS
* **Lenguajes:** Go, Python, TypeScript
* **Infraestructura:** Terraform (S3 + DynamoDB backend)
* **Arquitectura:** Hexagonal, Clean Architecture, Serverless
* **DevSecOps:** Git security, secrets handling, CI/CD-ready

---

## 📌 **Objetivos del Repositorio**

1. Construir un portafolio profesional demostrando dominio de AWS.
2. Mostrar prácticas reales de arquitectura y AppSec en la nube.
3. Documentar patrones reutilizables para proyectos serverless.
4. Crear una base de conocimiento pública para otros ingenieros.

---

## 🤝 **Colaboración**

Este repositorio está en evolución activa.
Pull requests, sugerencias y discusiones son bienvenidas siempre que respeten las **reglas de seguridad pública** del proyecto.

---

## 📫 **Contacto Profesional**

* Portafolio: [https://edgarmacias.com](https://edgarmacias.com)
* Laboratorios de Seguridad: [https://devcybsec.com](https://devcybsec.com)
* GitHub: [https://github.com/devcybsec](https://github.com/devcybsec)
