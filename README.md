# Trabajo Práctico Intermedio de Bases de Datos: Diseño de Esquema NoSQL
**Curso avanzado: Desarrollo Back-End**  
**Centro de e-Learning UTN BA**  
**Proyecto:** Jolie E-commerce  
**Alumna:** Valeria Villegas

---

## 1. Elección del Dominio

**Definición:** Plataforma web de comercio electrónico orientada a la venta minorista directa al consumidor final de indumentaria y accesorios, administrada por personal de catálogo y logística (`admin`) y consumida por clientes finales registrados (`cliente`) para
navegación, armado de carritos y concreción de pedidos.

---

## 2. Descripción del Modelo de Datos

El diseño de la base de datos en MongoDB se estructura entorno a las tres entidades fundamentales exigidas, complementadas con colecciones de soporte transaccional y fidelización para una arquitectura lista para producción:

### 2.1. Entidades Obligatorias del Sistema
* **`usuarios` (`users`)**: Representa a los actores del sistema. Incluye autenticación, datos personales, libreta de direcciones embebida y control de acceso basado en el atributo de rol (`admin` vs. `cliente`).
* **`productos` (`products` - Entidad Principal)**: Catálogo de artículos disponibles para la venta. Contiene atributos descriptivos, imágenes, estado operativo, un arreglo embebido de variantes (talles, colores y stock) y un campo de vinculación externa hacia 
la categoría.
* **`categorias` (`categories` - Entidad Referenciada)**: Estructura del catálogo alojada en una colección independiente. Mantiene una relación **1:N** con la colección `productos` mediante el identificador `id_categoria`.

### 2.2. Entidades Transaccionales y de Soporte
* **`ordenes` (`orders`)**: Registra la transacción de compra finalizada. Mantiene una referencia **N:1** con `usuarios` y almacena como subdocumentos embebidos tanto el *snapshot* de los artículos comprados como el historial de estados de entrega.
* **`carritos` (`carts`)**: Persistencia del estado temporal de compra por usuario (relación **1:1** con `usuarios`), optimizando escrituras de ítems pendientes.
* **`calificaciones` (`reviews`)**: Reseñas y puntajes otorgados por clientes verificados sobre productos específicos (relación **N:1** con `productos` y **N:1** con `usuarios`).
* **`favoritos` (`favorites`)**: Colección de persistencia de listas de deseos, asociando usuarios con productos de su interés.
* **`descuentos` (`discounts`)**: Cupones y reglas promocionales aplicables en el proceso de *checkout*.

> *Nota de entrega:* El diagrama de entidades y relaciones (formato gráfico) se adjunta en archivo separado conforme a las pautas de presentación.

---

## 3. Justificación de Embeber vs. Referenciar

El modelado en MongoDB exige ponderar la atomicidad de las operaciones, la frecuencia de lectura/escritura y el límite de 16 MB por documento. Se adoptaron las siguientes decisiones de arquitectura:

### 3.1. Por qué `categorias` se referencia y no se embebe en `productos`
* **Ciclo de vida independiente:** Las categorías existen conceptualmente sin necesidad de que haya productos asociados y requieren gestión CRUD autónoma por parte del administrador.
* **Consistencia e integridad:** Si se embebiera la categoría en cada producto, la actualización de un nombre o descripción exigiría mutaciones masivas e ineficientes sobre miles de registros (`multi: true`), aumentando el riesgo de inconsistencias ante escrituras 
parciales. La referencia normalizada (`id_categoria`) permite actualizaciones atómicas en un único documento.
* **Navegación y filtros eficientes:** Facilita la construcción de menús de navegación dinámicos en el frontend y filtros facetados mediante consultas indexadas simples.

### 3.2. Casos donde se optó por el Patrón de Embebido (Embedded Pattern)
* **`variantes` dentro de `productos`:**
  * **Cohesión de lectura:** Un cliente nunca consulta un talle o color aislado; la ficha de producto (*Product Detail Page*) siempre requiere la matriz completa de variantes y existencias en una sola operación de lectura.
  * **Concurrencia controlada:** Embeber permite actualizar el stock de una variante específica de forma atómica (`$inc` con filtros posicionales sobre el array), garantizando consistencia transaccional sin recurrir a bloqueos distribuidos.
* **`items` dentro de `ordenes` (Patrón Snapshot):**
  * **Inmutabilidad histórica y fiscal:** Los productos sufren modificaciones periódicas de precio, cambios de nombre o eventual descontinuación. Embeber los ítems comprados junto con su precio unitario, nombre y variante en el instante exacto de la orden "congela"
  la transacción, evitando que ajustes comerciales futuros corrompan el balance histórico de ventas.
* **`direccion_envio` dentro de `ordenes`:**
  * **Consistencia logística:** Si el usuario actualiza su libreta de direcciones en su perfil, la orden despachada en el pasado debe conservar irrevocablemente el domicilio físico donde fue entregada.
* **`direcciones` dentro de `usuarios`:**
  * **Límite acotado:** Un cliente maneja un número reducido de domicilios habituales (generalmente entre 1 y 5). Embeber este conjunto respeta las buenas prácticas de MongoDB (relación 1:Pocos) y elimina joins innecesarios en la carga del perfil.
* **`historial_estados` dentro de `ordenes`:**
  * **Trazabilidad directa:** Registrar cambios de estado logísticos dentro del mismo documento evita la creación de colecciones accesorias de seguimiento, centralizando toda la información operativa del pedido en una única consulta.

---

## 4. Índices Propuestos

Para asegurar un rendimiento óptimo de la API bajo alta concurrencia, se establecen los siguientes índices en la base de datos:

1. **`usuarios`: `{ "email": 1 }` (Unique)**
   * **Propósito:** Garantiza que no existan cuentas duplicadas a nivel motor de base de datos y optimiza la autenticación durante el login.
2. **`categorias`: `{ "nombre": 1 }` (Unique)**
   * **Propósito:** Evita redundancia en la taxonomía y acelera la búsqueda interna de categorías en el panel de gestión.
3. **`productos`: `{ "id_categoria": 1 }` (Simple)**
   * **Propósito:** Acelera drásticamente el filtrado de catálogo cuando los usuarios navegan por categorías específicas en la tienda.
4. **`productos`: `{ "activo": 1, "precio_base": 1 }` (Compuesto)**
   * **Propósito:** Optimiza el ordenamiento por precio dentro del catálogo visible al público, descartando ítems pausados.
5. **`ordenes`: `{ "id_usuario": 1, "fecha_creacion": -1 }` (Compuesto)**
   * **Propósito:** Agiliza la paginación cronológica descendente del historial de compras del cliente en su panel privado.
6. **`ordenes`: `{ "numero_orden": 1 }` (Unique)**
   * **Propósito:** Garantiza la unicidad del identificador alfanumérico visible de la compra para seguimiento y consultas de soporte.
7. **`carritos`: `{ "id_usuario": 1 }` (Unique)**
   * **Propósito:** Restringe a un único carrito activo por usuario registrado y agiliza su recuperación al iniciar sesión.
8. **`calificaciones`: `{ "id_producto": 1, "id_usuario": 1 }` (Compuesto Unique)**
   * **Propósito:** Impide que un mismo usuario emita múltiples reseñas sobre un mismo producto y acelera la carga de valoraciones en la página del artículo.
9. **`descuentos`: `{ "codigo": 1 }` (Unique)**
   * **Propósito:** Valida de manera inmediata la existencia del cupón ingresado en el *checkout* y asegura códigos alfanuméricos irrepetibles.

---

## 5. Documentos de Ejemplo

Documentos JSON representativos de cada colección con consistencia referencial en sus identificadores (`ObjectId`):

### 5.1. Colección: `categorias`
```json
[
  {
    "_id": {"$oid": "660000000000000000000001"},
    "nombre": "Remeras y Tops",
    "descripcion": "Prendas superiores básicas y de diseño",
    "imagen": "[https://res.cloudinary.com/jolie/cat-remeras.webp](https://res.cloudinary.com/jolie/cat-remeras.webp)",
    "activo": true,
    "fecha_creacion": {"$date": "2026-01-10T10:00:00.000Z"}
  },
  {
    "_id": {"$oid": "660000000000000000000002"},
    "nombre": "Abrigos",
    "descripcion": "Buzos, camperas y sweaters de temporada",
    "imagen": "[https://res.cloudinary.com/jolie/cat-abrigos.webp](https://res.cloudinary.com/jolie/cat-abrigos.webp)",
    "activo": true,
    "fecha_creacion": {"$date": "2026-01-10T10:05:00.000Z"}
  },
  {
    "_id": {"$oid": "660000000000000000000003"},
    "nombre": "Pantalones",
    "descripcion": "Denim, sastreros y joggers urbanos",
    "imagen": "[https://res.cloudinary.com/jolie/cat-pantalones.webp](https://res.cloudinary.com/jolie/cat-pantalones.webp)",
    "activo": true,
    "fecha_creacion": {"$date": "2026-01-10T10:10:00.000Z"}
  }
]
```
### 5.2. Colección: usuarios
```json

[
  {
    "_id": {"$oid": "661000000000000000000001"},
    "email": "admin@jolie-store.com",
    "rol": "admin",
    "nombre": "Valeria",
    "apellido": "Villegas",
    "telefono": "+5492964551122",
    "direcciones": [
      {
        "calle": "Av. San Martín 450",
        "ciudad": "Río Grande",
        "codigo_postal": "9420",
        "es_principal": true
      }
    ],
    "fecha_creacion": {"$date": "2026-02-01T08:00:00.000Z"}
  },
  {
    "_id": {"$oid": "661000000000000000000002"},
    "email": "marina.lopez@gmail.com",
    "rol": "cliente",
    "nombre": "Marina",
    "apellido": "López",
    "telefono": "+5492964883344",
    "direcciones": [
      {
        "calle": "Fagnano 1240, Dpto 2B",
        "ciudad": "Río Grande",
        "codigo_postal": "9420",
        "es_principal": true
      }
    ],
    "fecha_creacion": {"$date": "2026-02-15T14:20:00.000Z"}
  },
  {
    "_id": {"$oid": "661000000000000000000003"},
    "email": "federico.diaz@hotmail.com",
    "rol": "cliente",
    "nombre": "Federico",
    "apellido": "Díaz",
    "telefono": "+5491144556677",
    "direcciones": [
      {
        "calle": "Av. Cabildo 2200",
        "ciudad": "CABA",
        "codigo_postal": "1428",
        "es_principal": true
      },
      {
        "calle": "Defensa 830",
        "ciudad": "CABA",
        "codigo_postal": "1065",
        "es_principal": false
      }
    ],
    "fecha_creacion": {"$date": "2026-02-18T19:45:00.000Z"}
  }
]
```
### 5.3. Colección: productos
```json
[
  {
    "_id": {"$oid": "662000000000000000000001"},
    "nombre": "Buzo Oversize Noir",
    "descripcion": "Buzo unisex confeccionado en frisa pesada de algodón con terminaciones reforzadas.",
    "precio_base": 48500,
    "id_categoria": {"$oid": "660000000000000000000002"},
    "imagenes": [
      "https://res.cloudinary.com/jolie/buzo-noir-1.webp",
       "https://res.cloudinary.com/jolie/buzo-noir-2.webp"
    ],
    "variantes": [
      {
        "id_variante": {"$oid": "662000000000000000000101"},
        "talle": "M",
        "color": "Negro",
        "stock": 14
      },
      {
        "id_variante": {"$oid": "662000000000000000000102"},
        "talle": "L",
        "color": "Negro",
        "stock": 8
      }
    ],
    "activo": true,
    "fecha_creacion": {"$date": "2026-02-20T11:00:00.000Z"}
  },
  {
    "_id": {"$oid": "662000000000000000000002"},
    "nombre": "Remera Boxy Fit Basic",
    "descripcion": "Remera 100% algodón jersey peinado con corte suelto y cuello ribb al tono.",
    "precio_base": 24000,
    "id_categoria": {"$oid": "660000000000000000000001"},
    "imagenes": [
      "https://res.cloudinary.com/jolie/remera-boxy-white.webp"
    ],
    "variantes": [
      {
        "id_variante": {"$oid": "662000000000000000000201"},
        "talle": "S",
        "color": "Blanco",
        "stock": 20
      },
      {
        "id_variante": {"$oid": "662000000000000000000202"},
        "talle": "M",
        "color": "Blanco",
        "stock": 15
      }
    ],
    "activo": true,
    "fecha_creacion": {"$date": "2026-02-22T09:30:00.000Z"}
  },
  {
    "_id": {"$oid": "662000000000000000000003"},
    "nombre": "Pantalón Cargo Carpenter",
    "descripcion": "Pantalón de gabardina semipesada con múltiples bolsillos funcionales y tiro medio.",
    "precio_base": 56000,
    "id_categoria": {"$oid": "660000000000000000000003"},
    "imagenes": [
      "https://res.cloudinary.com/jolie/cargo-beige-1.webp"
    ],
    "variantes": [
      {
        "id_variante": {"$oid": "662000000000000000000301"},
        "talle": "40",
        "color": "Beige",
        "stock": 10
      },
      {
        "id_variante": {"$oid": "662000000000000000000302"},
        "talle": "42",
        "color": "Beige",
        "stock": 6
      }
    ],
    "activo": true,
    "fecha_creacion": {"$date": "2026-02-25T16:15:00.000Z"}
  }
]
```
### 5.4. Colección: ordenes
```json
[
  {
    "_id": {"$oid": "663000000000000000000001"},
    "numero_orden": "ORD-2026-0001",
    "id_usuario": {"$oid": "661000000000000000000002"},
    "items": [
      {
        "id_producto": {"$oid": "662000000000000000000001"},
        "id_variante": {"$oid": "662000000000000000000101"},
        "nombre_producto": "Buzo Oversize Noir",
        "talle": "M",
        "color": "Negro",
        "precio_unitario": 48500,
        "cantidad": 1
      },
      {
        "id_producto": {"$oid": "662000000000000000000002"},
        "id_variante": {"$oid": "662000000000000000000201"},
        "nombre_producto": "Remera Boxy Fit Basic",
        "talle": "S",
        "color": "Blanco",
        "precio_unitario": 24000,
        "cantidad": 2
      }
    ],
    "monto_total": 96500,
    "descuento_aplicado": 0,
    "estado": "entregado",
    "direccion_envio": {
      "calle": "Fagnano 1240, Dpto 2B",
      "ciudad": "Río Grande",
      "codigo_postal": "9420"
    },
    "metodo_pago": "MercadoPago",
    "id_transaccion_pago": "MP-TX-984729104",
    "historial_estados": [
      {
        "estado": "pendiente",
        "fecha": {"$date": "2026-03-01T12:00:00.000Z"},
        "nota": "Orden creada esperando acreditación",
        "modificado_por": {"$oid": "661000000000000000000002"}
      },
      {
        "estado": "pagado",
        "fecha": {"$date": "2026-03-01T12:05:00.000Z"},
        "nota": "Pago aprobado por webhook",
        "modificado_por": {"$oid": "661000000000000000000001"}
      },
      {
        "estado": "entregado",
        "fecha": {"$date": "2026-03-03T17:30:00.000Z"},
        "nota": "Recibido en domicilio",
        "modificado_por": {"$oid": "661000000000000000000001"}
      }
    ],
    "fecha_creacion": {"$date": "2026-03-01T12:00:00.000Z"},
    "fecha_pago": {"$date": "2026-03-01T12:05:00.000Z"}
  },
  {
    "_id": {"$oid": "663000000000000000000002"},
    "numero_orden": "ORD-2026-0002",
    "id_usuario": {"$oid": "661000000000000000000003"},
    "items": [
      {
        "id_producto": {"$oid": "662000000000000000000003"},
        "id_variante": {"$oid": "662000000000000000000301"},
        "nombre_producto": "Pantalón Cargo Carpenter",
        "talle": "40",
        "color": "Beige",
        "precio_unitario": 56000,
        "cantidad": 1
      }
    ],
    "monto_total": 50400,
    "descuento_aplicado": 5600,
    "estado": "enviado",
    "direccion_envio": {
      "calle": "Av. Cabildo 2200",
      "ciudad": "CABA",
      "codigo_postal": "1428"
    },
    "metodo_pago": "MercadoPago",
    "id_transaccion_pago": "MP-TX-883719201",
    "historial_estados": [
      {
        "estado": "pagado",
        "fecha": {"$date": "2026-03-04T10:15:00.000Z"},
        "nota": "Pago aprobado con cupón BIENVENIDA10",
        "modificado_por": {"$oid": "661000000000000000000001"}
      },
      {
        "estado": "enviado",
        "fecha": {"$date": "2026-03-05T14:00:00.000Z"},
        "nota": "Despachado con Correo Argentino",
        "modificado_por": {"$oid": "661000000000000000000001"}
      }
    ],
    "fecha_creacion": {"$date": "2026-03-04T10:10:00.000Z"},
    "fecha_pago": {"$date": "2026-03-04T10:15:00.000Z"}
  },
  {
    "_id": {"$oid": "663000000000000000000003"},
    "numero_orden": "ORD-2026-0003",
    "id_usuario": {"$oid": "661000000000000000000002"},
    "items": [
      {
        "id_producto": {"$oid": "662000000000000000000002"},
        "id_variante": {"$oid": "662000000000000000000202"},
        "nombre_producto": "Remera Boxy Fit Basic",
        "talle": "M",
        "color": "Blanco",
        "precio_unitario": 24000,
        "cantidad": 1
      }
    ],
    "monto_total": 24000,
    "descuento_aplicado": 0,
    "estado": "pendiente",
    "direccion_envio": {
      "calle": "Fagnano 1240, Dpto 2B",
      "ciudad": "Río Grande",
      "codigo_postal": "9420"
    },
    "metodo_pago": "Transferencia",
    "id_transaccion_pago": null,
    "historial_estados": [
      {
        "estado": "pendiente",
        "fecha": {"$date": "2026-03-08T18:30:00.000Z"},
        "nota": "A la espera de comprobante",
        "modificado_por": {"$oid": "661000000000000000000002"}
      }
    ],
    "fecha_creacion": {"$date": "2026-03-08T18:30:00.000Z"},
    "fecha_pago": null
  }
]
```
### 5.5. Colección: carritos
```json
[
  {
    "_id": {"$oid": "664000000000000000000001"},
    "id_usuario": {"$oid": "661000000000000000000002"},
    "items": [],
    "fecha_actualizacion": {"$date": "2026-03-08T18:30:00.000Z"}
  },
  {
    "_id": {"$oid": "664000000000000000000002"},
    "id_usuario": {"$oid": "661000000000000000000003"},
    "items": [
      {
        "id_producto": {"$oid": "662000000000000000000001"},
        "id_variante": {"$oid": "662000000000000000000102"},
        "cantidad": 1
      }
    ],
    "fecha_actualizacion": {"$date": "2026-03-09T14:10:00.000Z"}
  },
  {
    "_id": {"$oid": "664000000000000000000003"},
    "id_usuario": {"$oid": "661000000000000000000001"},
    "items": [],
    "fecha_actualizacion": {"$date": "2026-02-01T08:00:00.000Z"}
  }
]
```
### 5.6. Colección: calificaciones
```json
[
  {
    "_id": {"$oid": "665000000000000000000001"},
    "id_producto": {"$oid": "662000000000000000000001"},
    "id_usuario": {"$oid": "661000000000000000000002"},
    "puntuacion": 5,
    "titulo": "Excelente abrigo y calce",
    "comentario": "El buzo es de frisa pesada de verdad, muy abrigado y el talle M quedó perfecto.",
    "compra_verificada": true,
    "fecha_creacion": {"$date": "2026-03-04T11:00:00.000Z"}
  },
  {
    "_id": {"$oid": "665000000000000000000002"},
    "id_producto": {"$oid": "662000000000000000000002"},
    "id_usuario": {"$oid": "661000000000000000000002"},
    "puntuacion": 4,
    "titulo": "Buena tela pero cuello ajustado",
    "comentario": "El algodón es muy suave y resistente a los lavados. El corte boxy es impecable.",
    "compra_verificada": true,
    "fecha_creacion": {"$date": "2026-03-05T15:20:00.000Z"}
  },
  {
    "_id": {"$oid": "665000000000000000000003"},
    "id_producto": {"$oid": "662000000000000000000003"},
    "id_usuario": {"$oid": "661000000000000000000003"},
    "puntuacion": 5,
    "titulo": "Muy funcional",
    "comentario": "Los bolsillos carpenter son cómodos y la tela es gruesa. Recomendado.",
    "compra_verificada": true,
    "fecha_creacion": {"$date": "2026-03-07T18:40:00.000Z"}
  }
]
```
### 5.7. Colección: favoritos
```json
[
  {
    "_id": {"$oid": "667000000000000000000001"},
    "id_usuario": {"$oid": "661000000000000000000002"},
    "id_producto": {"$oid": "662000000000000000000003"},
    "fecha_agregado": {"$date": "2026-03-02T16:00:00.000Z"}
  },
  {
    "_id": {"$oid": "667000000000000000000002"},
    "id_usuario": {"$oid": "661000000000000000000003"},
    "id_producto": {"$oid": "662000000000000000000001"},
    "fecha_agregado": {"$date": "2026-03-03T10:15:00.000Z"}
  },
  {
    "_id": {"$oid": "667000000000000000000003"},
    "id_usuario": {"$oid": "661000000000000000000003"},
    "id_producto": {"$oid": "662000000000000000000002"},
    "fecha_agregado": {"$date": "2026-03-03T10:20:00.000Z"}
  }
]
```
5.8. Colección: descuentos
```json
[
  {
    "_id": {"$oid": "666000000000000000000001"},
    "codigo": "BIENVENIDA10",
    "tipo": "porcentaje",
    "valor": 10,
    "usos_maximos": 500,
    "cantidad_usos": 38,
    "valido_desde": {"$date": "2026-01-01T00:00:00.000Z"},
    "valido_hasta": {"$date": "2026-12-31T23:59:59.000Z"},
    "activo": true
  },
  {
    "_id": {"$oid": "666000000000000000000002"},
    "codigo": "HOTWEEK5000",
    "tipo": "fijo",
    "valor": 5000,
    "usos_maximos": 100,
    "cantidad_usos": 100,
    "valido_desde": {"$date": "2026-02-01T00:00:00.000Z"},
    "valido_hasta": {"$date": "2026-02-28T23:59:59.000Z"},
    "activo": false
  },
  {
    "_id": {"$oid": "666000000000000000000003"},
    "codigo": "JOLIEVIP",
    "tipo": "porcentaje",
    "valor": 15,
    "usos_maximos": 50,
    "cantidad_usos": 12,
    "valido_desde": {"$date": "2026-03-01T00:00:00.000Z"},
    "valido_hasta": {"$date": "2026-03-31T23:59:59.000Z"},
    "activo": true
  }
]

