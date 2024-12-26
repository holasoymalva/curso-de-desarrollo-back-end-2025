# Módulo 3: Bases de Datos y ORM

## 1. Fundamentos de Bases de Datos

### 1.1 SQL vs NoSQL
- **SQL (PostgreSQL)**:
  - Estructura fija (esquemas)
  - Relaciones definidas
  - ACID compliant
  - Ideal para datos estructurados

- **NoSQL (MongoDB)**:
  - Esquemas flexibles
  - Escalabilidad horizontal
  - Ideal para datos no estructurados
  - Alta disponibilidad

### 1.2 Modelado de Datos

#### MongoDB - Esquemas con Mongoose:
```javascript
// Ejemplo de modelado de datos para una aplicación de blog
const mongoose = require('mongoose');

// Schema de Usuario
const usuarioSchema = new mongoose.Schema({
    nombre: {
        type: String,
        required: [true, 'El nombre es requerido'],
        trim: true,
        minlength: [2, 'El nombre debe tener al menos 2 caracteres']
    },
    email: {
        type: String,
        required: [true, 'El email es requerido'],
        unique: true,
        lowercase: true,
        match: [/^\S+@\S+\.\S+$/, 'Por favor ingrese un email válido']
    },
    password: {
        type: String,
        required: [true, 'La contraseña es requerida'],
        minlength: 8,
        select: false // No se incluye en las consultas
    },
    posts: [{
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Post'
    }],
    createdAt: {
        type: Date,
        default: Date.now
    }
});

// Middleware pre-save para hashear password
usuarioSchema.pre('save', async function(next) {
    if (!this.isModified('password')) return next();
    
    this.password = await bcrypt.hash(this.password, 12);
    next();
});
```

#### PostgreSQL - Esquemas con Sequelize:
```javascript
// Modelos con Sequelize
const { Model, DataTypes } = require('sequelize');

class Usuario extends Model {}
Usuario.init({
    id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    nombre: {
        type: DataTypes.STRING,
        allowNull: false,
        validate: {
            len: [2, 50]
        }
    },
    email: {
        type: DataTypes.STRING,
        allowNull: false,
        unique: true,
        validate: {
            isEmail: true
        }
    },
    password: {
        type: DataTypes.STRING,
        allowNull: false,
        validate: {
            len: [8, 100]
        }
    }
}, {
    sequelize,
    modelName: 'Usuario',
    hooks: {
        beforeSave: async (usuario) => {
            if (usuario.changed('password')) {
                usuario.password = await bcrypt.hash(usuario.password, 12);
            }
        }
    }
});
```

## 2. MongoDB y Mongoose

### 2.1 Conexión y Configuración
```javascript
// config/database.js
const mongoose = require('mongoose');

const conectarDB = async () => {
    try {
        const conn = await mongoose.connect(process.env.MONGO_URI, {
            useNewUrlParser: true,
            useUnifiedTopology: true,
            useCreateIndex: true,
            useFindAndModify: false
        });
        
        console.log(`MongoDB conectado: ${conn.connection.host}`);
    } catch (error) {
        console.error(`Error: ${error.message}`);
        process.exit(1);
    }
};

module.exports = conectarDB;
```

### 2.2 Operaciones CRUD
```javascript
// Controladores de Usuario
const Usuario = require('../models/usuario.model');

// Crear
const crearUsuario = async (req, res) => {
    try {
        const usuario = await Usuario.create(req.body);
        usuario.password = undefined; // No enviar password en respuesta
        res.status(201).json(usuario);
    } catch (error) {
        if (error.code === 11000) { // Error de duplicado
            res.status(400).json({
                mensaje: 'El email ya está registrado'
            });
        } else {
            res.status(400).json({
                mensaje: error.message
            });
        }
    }
};

// Leer (con población de referencias)
const obtenerUsuarioConPosts = async (req, res) => {
    try {
        const usuario = await Usuario
            .findById(req.params.id)
            .populate({
                path: 'posts',
                select: 'titulo contenido createdAt',
                options: { sort: { createdAt: -1 } }
            });
            
        if (!usuario) {
            return res.status(404).json({
                mensaje: 'Usuario no encontrado'
            });
        }
        
        res.json(usuario);
    } catch (error) {
        res.status(500).json({
            mensaje: error.message
        });
    }
};

// Actualizar
const actualizarUsuario = async (req, res) => {
    try {
        const usuario = await Usuario.findByIdAndUpdate(
            req.params.id,
            req.body,
            {
                new: true, // Retorna el documento actualizado
                runValidators: true // Ejecuta las validaciones del schema
            }
        );
        
        if (!usuario) {
            return res.status(404).json({
                mensaje: 'Usuario no encontrado'
            });
        }
        
        res.json(usuario);
    } catch (error) {
        res.status(400).json({
            mensaje: error.message
        });
    }
};

// Eliminar
const eliminarUsuario = async (req, res) => {
    try {
        const usuario = await Usuario.findByIdAndDelete(req.params.id);
        
        if (!usuario) {
            return res.status(404).json({
                mensaje: 'Usuario no encontrado'
            });
        }
        
        res.json({
            mensaje: 'Usuario eliminado exitosamente'
        });
    } catch (error) {
        res.status(500).json({
            mensaje: error.message
        });
    }
};
```

## 3. PostgreSQL y Sequelize

### 3.1 Conexión y Configuración
```javascript
// config/database.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(process.env.DB_NAME, process.env.DB_USER, process.env.DB_PASSWORD, {
    host: process.env.DB_HOST,
    dialect: 'postgres',
    logging: false, // Desactivar logging SQL
    pool: {
        max: 5,
        min: 0,
        acquire: 30000,
        idle: 10000
    }
});

const conectarDB = async () => {
    try {
        await sequelize.authenticate();
        console.log('Conexión a PostgreSQL establecida.');
        
        // Sincronizar modelos con la base de datos
        await sequelize.sync({ alter: true });
        console.log('Modelos sincronizados.');
    } catch (error) {
        console.error('Error conectando a PostgreSQL:', error);
        process.exit(1);
    }
};

module.exports = { sequelize, conectarDB };
```

### 3.2 Operaciones CRUD y Relaciones
```javascript
// Definición de relaciones
Usuario.hasMany(Post, {
    foreignKey: 'usuarioId',
    as: 'posts'
});

Post.belongsTo(Usuario, {
    foreignKey: 'usuarioId',
    as: 'autor'
});

// Controladores
const crearPost = async (req, res) => {
    try {
        const post = await Post.create({
            ...req.body,
            usuarioId: req.usuario.id // De middleware de auth
        });
        
        res.status(201).json(post);
    } catch (error) {
        res.status(400).json({
            mensaje: error.message
        });
    }
};

// Consultas complejas
const obtenerPostsPopulares = async (req, res) => {
    try {
        const posts = await Post.findAll({
            include: [{
                model: Usuario,
                as: 'autor',
                attributes: ['nombre', 'email']
            }],
            where: {
                likes: {
                    [Op.gte]: 10 // Más de 10 likes
                }
            },
            order: [
                ['createdAt', 'DESC']
            ],
            limit: 10
        });
        
        res.json(posts);
    } catch (error) {
        res.status(500).json({
            mensaje: error.message
        });
    }
};
```

## 4. Patrones y Mejores Prácticas

### 4.1 Repositorio Pattern
```javascript
// repositories/base.repository.js
class BaseRepository {
    constructor(model) {
        this.model = model;
    }
    
    async crear(data) {
        return await this.model.create(data);
    }
    
    async encontrarPorId(id) {
        return await this.model.findById(id);
    }
    
    async encontrarTodos(filtro = {}) {
        return await this.model.find(filtro);
    }
    
    async actualizar(id, data) {
        return await this.model.findByIdAndUpdate(id, data, { new: true });
    }
    
    async eliminar(id) {
        return await this.model.findByIdAndDelete(id);
    }
}

// repositories/usuario.repository.js
class UsuarioRepository extends BaseRepository {
    constructor() {
        super(Usuario);
    }
    
    async encontrarPorEmail(email) {
        return await this.model.findOne({ email });
    }
    
    async encontrarConPosts(id) {
        return await this.model
            .findById(id)
            .populate('posts');
    }
}
```

### 4.2 Servicio Pattern
```javascript
// services/auth.service.js
class AuthService {
    constructor(usuarioRepository) {
        this.usuarioRepository = usuarioRepository;
    }
    
    async registrar(userData) {
        const usuarioExistente = await this.usuarioRepository.encontrarPorEmail(userData.email);
        
        if (usuarioExistente) {
            throw new Error('Email ya registrado');
        }
        
        const usuario = await this.usuarioRepository.crear(userData);
        return this.generarToken(usuario);
    }
    
    async login(email, password) {
        const usuario = await this.usuarioRepository.encontrarPorEmail(email);
        
        if (!usuario || !(await usuario.compararPassword(password))) {
            throw new Error('Credenciales inválidas');
        }
        
        return this.generarToken(usuario);
    }
    
    generarToken(usuario) {
        return jwt.sign(
            { id: usuario._id },
            process.env.JWT_SECRET,
            { expiresIn: '1d' }
        );
    }
}
```

## Proyecto Práctico: Sistema de Blog con Múltiples Bases de Datos

Este proyecto demuestra el uso de MongoDB y PostgreSQL en la misma aplicación:

```javascript
// Configuración de bases de datos
const { conectarMongo } = require('./config/mongodb');
const { conectarPostgres } = require('./config/postgres');

// MongoDB para contenido (posts, comentarios)
const Post = require('./models/mongo/post.model');
const Comentario = require('./models/mongo/comentario.model');

// PostgreSQL para usuarios y analytics
const Usuario = require('./models/postgres/usuario.model');
const Analytics = require('./models/postgres/analytics.model');

// Servicio que interactúa con ambas bases de datos
class BlogService {
    async crearPost(usuarioId, postData) {
        // Verificar usuario en PostgreSQL
        const usuario = await Usuario.findByPk(usuarioId);
        if (!usuario) throw new Error('Usuario no encontrado');
        
        // Crear post en MongoDB
        const post = await Post.create({
            ...postData,
            autorId: usuarioId
        });
        
        // Registrar analítica en PostgreSQL
        await Analytics.create({
            tipo: 'POST_CREADO',
            usuarioId,
            metadata: { postId: post._id }
        });
        
        return post;
    }
    
    async obtenerEstadisticasUsuario(usuarioId) {
        // Datos de PostgreSQL
        const usuario = await Usuario.findByPk(usuarioId);
        const analytics = await Analytics.findAll({
            where: { usuarioId }
        });
        
        // Datos de MongoDB
        const posts = await Post.countDocuments({ autorId: usuarioId });
        const comentarios = await Comentario.countDocuments({ autorId: usuarioId });
        
        return {
            usuario,
            estadisticas: {
                totalPosts: posts,
                totalComentarios: comentarios,
                analytics: analytics.length
            }
        };
    }
}
```

Este proyecto demuestra:
- Uso de múltiples bases de datos
- Patrones de diseño (Repository, Service)
- Manejo de transacciones
- Sincronización de datos entre bases de datos
- Análisis y métricas
- Arquitectura escalable
