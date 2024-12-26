# Módulo 2: APIs RESTful y Express.js

## 1. Fundamentos de HTTP y REST

### 1.1 Protocolo HTTP
- **Métodos HTTP**:
  - GET: Obtener recursos
  - POST: Crear recursos
  - PUT: Actualizar recursos (completo)
  - PATCH: Actualizar recursos (parcial)
  - DELETE: Eliminar recursos

- **Códigos de Estado HTTP**:
```javascript
// Ejemplos de uso común de códigos HTTP
const statusCodes = {
    // Éxito
    200: 'OK - Petición exitosa',
    201: 'Created - Recurso creado exitosamente',
    204: 'No Content - Petición exitosa sin contenido de respuesta',
    
    // Redirecciones
    301: 'Moved Permanently - Recurso movido permanentemente',
    302: 'Found - Redirección temporal',
    
    // Errores del Cliente
    400: 'Bad Request - Petición mal formada',
    401: 'Unauthorized - No autorizado',
    403: 'Forbidden - Prohibido',
    404: 'Not Found - Recurso no encontrado',
    409: 'Conflict - Conflicto con el estado actual',
    
    // Errores del Servidor
    500: 'Internal Server Error - Error interno del servidor',
    502: 'Bad Gateway - Error de gateway',
    503: 'Service Unavailable - Servicio no disponible'
};
```

### 1.2 Principios REST
1. **Arquitectura Cliente-Servidor**
2. **Sin Estado (Stateless)**
3. **Cacheable**
4. **Sistema por Capas**
5. **Interfaz Uniforme**

Ejemplo de estructura de URLs RESTful:
```javascript
// Estructura de endpoints para un recurso 'usuarios'
const endpoints = {
    'GET /usuarios':         'Obtener lista de usuarios',
    'GET /usuarios/:id':     'Obtener un usuario específico',
    'POST /usuarios':        'Crear un nuevo usuario',
    'PUT /usuarios/:id':     'Actualizar un usuario completo',
    'PATCH /usuarios/:id':   'Actualizar campos específicos',
    'DELETE /usuarios/:id':  'Eliminar un usuario'
};
```

## 2. Express.js Fundamental

### 2.1 Configuración Básica
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

// Middleware para parsear JSON
app.use(express.json());

// Middleware para parsear URL-encoded bodies
app.use(express.urlencoded({ extended: true }));

// Middleware de logging básico
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url}`);
    next();
});

// Ruta básica
app.get('/', (req, res) => {
    res.json({ mensaje: 'API funcionando correctamente' });
});

// Manejo de errores 404
app.use((req, res) => {
    res.status(404).json({ error: 'Ruta no encontrada' });
});

app.listen(port, () => {
    console.log(`Servidor corriendo en puerto ${port}`);
});
```

### 2.2 Routing Avanzado
```javascript
// usuarios.routes.js
const express = require('express');
const router = express.Router();

// Middleware específico para esta ruta
router.use((req, res, next) => {
    console.log('Time:', Date.now());
    next();
});

// Parámetros de ruta
router.get('/:id', (req, res) => {
    const { id } = req.params;
    res.json({ mensaje: `Obteniendo usuario ${id}` });
});

// Parámetros de consulta
router.get('/', (req, res) => {
    const { limite = 10, pagina = 1 } = req.query;
    res.json({
        limite: parseInt(limite),
        pagina: parseInt(pagina)
    });
});

module.exports = router;

// app.js
const usuariosRouter = require('./routes/usuarios.routes');
app.use('/api/usuarios', usuariosRouter);
```

## 3. Middleware en Express

### 3.1 Tipos de Middleware
```javascript
// 1. Middleware de Aplicación
app.use((req, res, next) => {
    req.requestTime = Date.now();
    next();
});

// 2. Middleware de Router
router.use((req, res, next) => {
    console.log('Router Middleware');
    next();
});

// 3. Middleware de Manejo de Errores
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({
        error: 'Algo salió mal!',
        message: err.message
    });
});

// 4. Middleware de Terceros
const morgan = require('morgan');
app.use(morgan('dev'));
```

### 3.2 Middleware Personalizado
```javascript
// Middleware de autenticación
const autenticarToken = (req, res, next) => {
    const token = req.headers['authorization'];
    
    if (!token) {
        return res.status(401).json({
            error: 'Token no proporcionado'
        });
    }
    
    try {
        // Verificar token
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.usuario = decoded;
        next();
    } catch (error) {
        res.status(401).json({
            error: 'Token inválido'
        });
    }
};

// Middleware de validación
const validarUsuario = (req, res, next) => {
    const { nombre, email, password } = req.body;
    
    if (!nombre || !email || !password) {
        return res.status(400).json({
            error: 'Todos los campos son requeridos'
        });
    }
    
    if (password.length < 6) {
        return res.status(400).json({
            error: 'La contraseña debe tener al menos 6 caracteres'
        });
    }
    
    next();
};
```

## 4. Manejo de Errores

### 4.1 Errores Síncronos y Asíncronos
```javascript
// Clase de error personalizada
class APIError extends Error {
    constructor(statusCode, message) {
        super(message);
        this.statusCode = statusCode;
        this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
        this.isOperational = true;

        Error.captureStackTrace(this, this.constructor);
    }
}

// Middleware de captura de errores
const errorHandler = (err, req, res, next) => {
    err.statusCode = err.statusCode || 500;
    err.status = err.status || 'error';

    if (process.env.NODE_ENV === 'development') {
        res.status(err.statusCode).json({
            status: err.status,
            error: err,
            message: err.message,
            stack: err.stack
        });
    } else {
        // Producción: no enviar detalles del error
        res.status(err.statusCode).json({
            status: err.status,
            message: err.message
        });
    }
};

// Controlador con manejo de errores
const obtenerUsuario = async (req, res, next) => {
    try {
        const usuario = await Usuario.findById(req.params.id);
        
        if (!usuario) {
            throw new APIError(404, 'Usuario no encontrado');
        }
        
        res.json(usuario);
    } catch (error) {
        next(error);
    }
};
```

## Proyecto Práctico: API RESTful de Blog

Este proyecto integra todos los conceptos aprendidos:

```javascript
// app.js
const express = require('express');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');
const mongoSanitize = require('express-mongo-sanitize');
const xss = require('xss-clean');
const hpp = require('hpp');

const app = express();

// Middleware de Seguridad
app.use(helmet());  // Headers de seguridad
app.use(mongoSanitize());  // Contra inyección NoSQL
app.use(xss());  // Contra XSS
app.use(hpp());  // Contra contaminación de parámetros

// Rate limiting
const limiter = rateLimit({
    max: 100,  // Límite de peticiones
    windowMs: 60 * 60 * 1000,  // Ventana de tiempo (1 hora)
    message: 'Demasiadas peticiones desde esta IP'
});
app.use('/api', limiter);

// Rutas del blog
app.use('/api/posts', require('./routes/posts.routes'));
app.use('/api/comentarios', require('./routes/comentarios.routes'));
app.use('/api/usuarios', require('./routes/usuarios.routes'));

// Manejo de errores global
app.use(errorHandler);

// posts.routes.js
router.get('/', async (req, res, next) => {
    try {
        const posts = await Post.find()
            .populate('autor', 'nombre email')
            .sort('-createdAt')
            .limit(parseInt(req.query.limite) || 10)
            .skip((parseInt(req.query.pagina) - 1) * 10);

        res.json({
            resultados: posts.length,
            data: posts
        });
    } catch (error) {
        next(error);
    }
});

router.post('/', autenticarToken, validarPost, async (req, res, next) => {
    try {
        const post = await Post.create({
            titulo: req.body.titulo,
            contenido: req.body.contenido,
            autor: req.usuario.id
        });

        res.status(201).json(post);
    } catch (error) {
        next(error);
    }
});
```

El proyecto incluye:
- Arquitectura RESTful completa
- Middleware de seguridad
- Autenticación y autorización
- Validación de datos
- Manejo de errores robusto
- Rate limiting
- Relaciones entre recursos (posts, comentarios, usuarios)
- Paginación y filtrado
- Población de referencias
- Sanitización de datos
