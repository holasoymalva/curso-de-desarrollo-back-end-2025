# Módulo 4: Autenticación y Seguridad

## 1. JWT (JSON Web Tokens)

### 1.1 Fundamentos de JWT
- **Estructura de un JWT**:
  - Header (Algoritmo y tipo)
  - Payload (Datos)
  - Signature (Firma)

```javascript
// Implementación básica de JWT
const jwt = require('jsonwebtoken');

class JWTService {
    constructor() {
        this.secretKey = process.env.JWT_SECRET;
        this.expiresIn = '24h';
    }

    generateToken(payload) {
        return jwt.sign(payload, this.secretKey, {
            expiresIn: this.expiresIn
        });
    }

    verifyToken(token) {
        try {
            return jwt.verify(token, this.secretKey);
        } catch (error) {
            if (error instanceof jwt.TokenExpiredError) {
                throw new Error('Token expirado');
            }
            if (error instanceof jwt.JsonWebTokenError) {
                throw new Error('Token inválido');
            }
            throw error;
        }
    }

    decodeToken(token) {
        return jwt.decode(token, { complete: true });
    }
}
```

### 1.2 Middleware de Autenticación
```javascript
// middleware/auth.middleware.js
const authMiddleware = async (req, res, next) => {
    try {
        // Obtener token del header
        const token = req.headers.authorization?.split(' ')[1];
        
        if (!token) {
            return res.status(401).json({
                mensaje: 'No se proporcionó token de autenticación'
            });
        }

        // Verificar token
        const decoded = jwtService.verifyToken(token);
        
        // Buscar usuario en la base de datos
        const usuario = await Usuario.findById(decoded.id).select('-password');
        
        if (!usuario) {
            return res.status(401).json({
                mensaje: 'Token no válido - usuario no existe'
            });
        }

        // Agregar usuario a la request
        req.usuario = usuario;
        next();
    } catch (error) {
        res.status(401).json({
            mensaje: 'Token no válido',
            error: error.message
        });
    }
};
```

## 2. Autenticación con Passport

### 2.1 Estrategias de Passport
```javascript
// config/passport.js
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const GoogleStrategy = require('passport-google-oauth20').Strategy;

// Estrategia Local
passport.use(new LocalStrategy({
    usernameField: 'email',
    passwordField: 'password'
}, async (email, password, done) => {
    try {
        const usuario = await Usuario.findOne({ email });
        
        if (!usuario) {
            return done(null, false, { mensaje: 'Email no registrado' });
        }

        const isMatch = await usuario.compararPassword(password);
        
        if (!isMatch) {
            return done(null, false, { mensaje: 'Contraseña incorrecta' });
        }

        return done(null, usuario);
    } catch (error) {
        return done(error);
    }
}));

// Estrategia Google OAuth2.0
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
    try {
        let usuario = await Usuario.findOne({ googleId: profile.id });
        
        if (!usuario) {
            usuario = await Usuario.create({
                googleId: profile.id,
                nombre: profile.displayName,
                email: profile.emails[0].value,
                imagen: profile.photos[0].value
            });
        }
        
        return done(null, usuario);
    } catch (error) {
        return done(error);
    }
}));

// Serialización
passport.serializeUser((usuario, done) => {
    done(null, usuario.id);
});

// Deserialización
passport.deserializeUser(async (id, done) => {
    try {
        const usuario = await Usuario.findById(id);
        done(null, usuario);
    } catch (error) {
        done(error);
    }
});
```

### 2.2 Rutas de Autenticación
```javascript
// routes/auth.routes.js
const router = express.Router();

// Login local
router.post('/login', passport.authenticate('local'), (req, res) => {
    const token = jwtService.generateToken({ id: req.user.id });
    res.json({ token });
});

// Google OAuth
router.get('/google',
    passport.authenticate('google', { scope: ['profile', 'email'] })
);

router.get('/google/callback',
    passport.authenticate('google', { failureRedirect: '/login' }),
    (req, res) => {
        const token = jwtService.generateToken({ id: req.user.id });
        res.redirect(`/auth-success?token=${token}`);
    }
);
```

## 3. Seguridad en APIs

### 3.1 Protección contra Ataques Comunes
```javascript
// middleware/security.middleware.js
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const xss = require('xss-clean');
const hpp = require('hpp');

// Configuración de seguridad
const configurarSeguridad = (app) => {
    // Headers de seguridad
    app.use(helmet());

    // Rate limiting
    const limiter = rateLimit({
        max: 100, // Límite de peticiones
        windowMs: 60 * 60 * 1000, // 1 hora
        message: 'Demasiadas peticiones desde esta IP'
    });
    app.use('/api', limiter);

    // Sanitización de datos
    app.use(mongoSanitize()); // Previene inyección NoSQL
    app.use(xss()); // Limpia input de XSS

    // Prevención de Parameter Pollution
    app.use(hpp({
        whitelist: [
            'precio',
            'fecha',
            'rating'
        ]
    }));

    // CORS
    app.use(cors({
        origin: process.env.FRONTEND_URL,
        credentials: true
    }));
};
```

### 3.2 Validación y Sanitización
```javascript
// middleware/validator.middleware.js
const { body, validationResult } = require('express-validator');

const validarRegistro = [
    body('nombre')
        .trim()
        .isLength({ min: 2 })
        .escape()
        .withMessage('El nombre debe tener al menos 2 caracteres'),
    
    body('email')
        .isEmail()
        .normalizeEmail()
        .withMessage('Email no válido'),
    
    body('password')
        .isLength({ min: 8 })
        .withMessage('La contraseña debe tener al menos 8 caracteres')
        .matches(/\d/)
        .withMessage('La contraseña debe contener al menos un número')
        .matches(/[A-Z]/)
        .withMessage('La contraseña debe contener al menos una mayúscula'),
    
    (req, res, next) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        next();
    }
];
```

## 4. Manejo de Roles y Permisos

### 4.1 RBAC (Role-Based Access Control)
```javascript
// models/rol.model.js
const rolSchema = new mongoose.Schema({
    nombre: {
        type: String,
        required: true,
        unique: true
    },
    permisos: [{
        type: String,
        enum: ['crear', 'leer', 'actualizar', 'eliminar']
    }]
});

// middleware/rbac.middleware.js
const verificarPermisos = (permisosRequeridos) => {
    return async (req, res, next) => {
        try {
            const usuario = req.usuario;
            const rol = await Rol.findById(usuario.rolId);
            
            const tienePermiso = permisosRequeridos.every(permiso => 
                rol.permisos.includes(permiso)
            );
            
            if (!tienePermiso) {
                return res.status(403).json({
                    mensaje: 'No tiene permisos suficientes'
                });
            }
            
            next();
        } catch (error) {
            res.status(500).json({
                mensaje: 'Error al verificar permisos'
            });
        }
    };
};
```

### 4.2 ACL (Access Control List)
```javascript
// utils/acl.js
const ACL = {
    roles: {
        admin: {
            can: ['*']
        },
        editor: {
            can: ['crear_post', 'editar_post', 'eliminar_post'],
            inherits: ['usuario']
        },
        usuario: {
            can: ['leer_post', 'crear_comentario']
        }
    },
    
    check(rol, permiso) {
        const permisos = this.roles[rol]?.can || [];
        return permisos.includes('*') || permisos.includes(permiso);
    }
};

// middleware/acl.middleware.js
const verificarACL = (permiso) => {
    return (req, res, next) => {
        const rol = req.usuario.rol;
        
        if (!ACL.check(rol, permiso)) {
            return res.status(403).json({
                mensaje: 'Acceso denegado'
            });
        }
        
        next();
    };
};
```

## Proyecto Práctico: Sistema de Autenticación Multiproveedor

```javascript
// services/auth.service.js
class AuthService {
    constructor() {
        this.jwtService = new JWTService();
    }

    async login(provider, credentials) {
        try {
            let usuario;

            switch (provider) {
                case 'local':
                    usuario = await this.loginLocal(credentials);
                    break;
                case 'google':
                    usuario = await this.loginGoogle(credentials);
                    break;
                case 'facebook':
                    usuario = await this.loginFacebook(credentials);
                    break;
                default:
                    throw new Error('Proveedor de autenticación no soportado');
            }

            const token = this.jwtService.generateToken({
                id: usuario.id,
                rol: usuario.rol
            });

            await this.registrarSesion(usuario.id, {
                provider,
                userAgent: credentials.userAgent,
                ip: credentials.ip
            });

            return {
                token,
                usuario: this.filtrarDatosUsuario(usuario)
            };
        } catch (error) {
            throw new Error(`Error en autenticación: ${error.message}`);
        }
    }

    async registrarSesion(usuarioId, datos) {
        await Sesion.create({
            usuarioId,
            ...datos,
            fechaInicio: new Date()
        });
    }

    filtrarDatosUsuario(usuario) {
        const { password, ...datosPublicos } = usuario.toObject();
        return datosPublicos;
    }
}

// controllers/auth.controller.js
class AuthController {
    constructor() {
        this.authService = new AuthService();
    }

    async login(req, res) {
        try {
            const { provider } = req.params;
            const credentials = {
                ...req.body,
                userAgent: req.headers['user-agent'],
                ip: req.ip
            };

            const resultado = await this.authService.login(provider, credentials);
            res.json(resultado);
        } catch (error) {
            res.status(401).json({
                mensaje: error.message
            });
        }
    }
}

// routes/auth.routes.js
router.post('/login/:provider', 
    validarCredenciales,
    rateLimiter,
    authController.login
);
```

Este proyecto demuestra:
- Múltiples proveedores de autenticación
- Sistema de sesiones
- Rate limiting
- Validación de credenciales
- Registro de actividad
- Manejo de tokens
- Sistema de roles y permisos
- Seguridad robusta
