# Comparativa: Clean Architecture vs Hexagonal vs Vertical Slice

## **Clean Architecture**

### Concepto principal
Separación en capas concéntricas donde las dependencias apuntan hacia adentro (hacia el dominio).

### Estructura
- **Entidades** (centro): Lógica de negocio empresarial
- **Casos de Uso**: Lógica de aplicación específica
- **Adaptadores de Interface**: Controladores, presentadores, gateways
- **Frameworks y Drivers** (exterior): UI, DB, web, dispositivos

### Ventajas
- Independencia del framework, UI y base de datos
- Altamente testeable
- Lógica de negocio bien aislada
- Buena para sistemas complejos y de larga duración

### Desventajas
- Puede ser excesiva para aplicaciones simples
- Mucho boilerplate y abstracción
- Curva de aprendizaje pronunciada
- Muchos archivos y carpetas

---

## **Hexagonal Architecture (Puertos y Adaptadores)**

### Concepto principal
El dominio está en el centro, rodeado de puertos (interfaces) y adaptadores que conectan con el mundo exterior.

### Estructura
- **Núcleo/Dominio**: Lógica de negocio pura
- **Puertos**: Interfaces que definen contratos
- **Adaptadores**: Implementaciones concretas (REST, DB, messaging)

### Ventajas
- Muy flexible y extensible
- Excelente para testing (puedes mockear adaptadores)
- Permite cambiar tecnologías fácilmente
- Clara separación entre lógica de negocio e infraestructura

### Desventajas
- Requiere disciplina del equipo
- Puede llevar a sobre-ingeniería
- Más archivos y abstracciones que mantener

---

## **Vertical Slice Architecture**

### Concepto principal
Organizar el código por features/casos de uso completos, cortando verticalmente todas las capas.

### Estructura
```
/Features
  /CreateUser
    - CreateUserCommand.cs
    - CreateUserHandler.cs
    - CreateUserValidator.cs
    - CreateUserEndpoint.cs
  /GetUser
    - GetUserQuery.cs
    - GetUserHandler.cs
```

### Ventajas
- **Alta cohesión**: Todo lo relacionado con una feature está junto
- Fácil de entender y navegar
- Menos acoplamiento entre features
- Ideal para agregar/quitar funcionalidades
- Menos código compartido = menos riesgo al hacer cambios
- Escalable en equipos (cada equipo puede tener sus slices)

### Desventajas
- Puede haber duplicación de código
- Menos reutilización de componentes
- Requiere refactorización cuando surgen patrones comunes
- Puede ser confuso si no se establece una estructura clara

---

## **Comparación Directa**

| Aspecto | Clean Architecture | Hexagonal | Vertical Slice |
|---------|-------------------|-----------|----------------|
| **Organización** | Por capas técnicas | Por puertos/adaptadores | Por features |
| **Complejidad inicial** | Alta | Media-Alta | Baja-Media |
| **Escalabilidad** | Excelente | Excelente | Muy buena |
| **Curva aprendizaje** | Pronunciada | Media | Suave |
| **Duplicación código** | Baja | Baja | Puede ser alta |
| **Acoplamiento** | Bajo entre capas | Muy bajo | Bajo entre slices |
| **Mejor para** | Sistemas complejos | Sistemas que cambian de tecnología | Equipos ágiles, MVPs |

---

## **¿Cuándo Usar Cada Una?**

### Clean Architecture
- Proyectos empresariales grandes
- Cuando la lógica de negocio es compleja y crítica
- Sistemas que vivirán muchos años

### Hexagonal Architecture
- Cuando necesitas flexibilidad para cambiar proveedores/tecnologías
- Microservicios
- Cuando el testing es prioritario

### Vertical Slice Architecture
- Startups y MVPs
- Equipos que trabajan en features independientes
- Cuando la velocidad de desarrollo es clave
- Aplicaciones CRUD con reglas de negocio moderadas

---

## **Nota Importante**

Estas arquitecturas no son mutuamente excluyentes. Puedes combinarlas, por ejemplo: usar Vertical Slices donde cada slice internamente sigue principios hexagonales.

---

## **Recursos Adicionales**

- **Clean Architecture**: "Clean Architecture" por Robert C. Martin
- **Hexagonal Architecture**: Artículos de Alistair Cockburn
- **Vertical Slice**: Jimmy Bogard (creator de MediatR) tiene excelentes charlas sobre este patrón

---

*Documento generado el 29 de octubre de 2025*
