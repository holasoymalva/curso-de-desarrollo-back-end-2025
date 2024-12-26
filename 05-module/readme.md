# Módulo 5: Testing y Calidad de Código

## 1. Unit Testing con Jest

### 1.1 Configuración Básica
```javascript
// jest.config.js
module.exports = {
    testEnvironment: 'node',
    verbose: true,
    collectCoverage: true,
    coverageDirectory: 'coverage',
    coveragePathIgnorePatterns: [
        '/node_modules/',
        '/tests/'
    ],
    testMatch: ['**/*.test.js']
};

// package.json
{
    "scripts": {
        "test": "jest",
        "test:watch": "jest --watch",
        "test:coverage": "jest --coverage"
    }
}
```

### 1.2 Escribiendo Tests Unitarios
```javascript
// services/calculadora.service.js
class CalculadoraService {
    sumar(a, b) {
        if (typeof a !== 'number' || typeof b !== 'number') {
            throw new Error('Los argumentos deben ser números');
        }
        return a + b;
    }

    dividir(a, b) {
        if (b === 0) {
            throw new Error('No se puede dividir por cero');
        }
        return a / b;
    }
}

// tests/calculadora.test.js
const CalculadoraService = require('../services/calculadora.service');

describe('CalculadoraService', () => {
    let calculadora;

    beforeEach(() => {
        calculadora = new CalculadoraService();
    });

    describe('sumar', () => {
        test('debe sumar dos números correctamente', () => {
            expect(calculadora.sumar(2, 3)).toBe(5);
        });

        test('debe lanzar error si los argumentos no son números', () => {
            expect(() => calculadora.sumar('2', 3)).toThrow('Los argumentos deben ser números');
        });
    });

    describe('dividir', () => {
        test('debe dividir dos números correctamente', () => {
            expect(calculadora.dividir(6, 2)).toBe(3);
        });

        test('debe lanzar error al dividir por cero', () => {
            expect(() => calculadora.dividir(6, 0)).toThrow('No se puede dividir por cero');
        });
    });
});
```

### 1.3 Mocking y Spies
```javascript
// services/usuario.service.js
class UsuarioService {
    constructor(usuarioRepository, emailService) {
        this.usuarioRepository = usuarioRepository;
        this.emailService = emailService;
    }

    async crearUsuario(datos) {
        const usuario = await this.usuarioRepository.crear(datos);
        await this.emailService.enviarBienvenida(usuario.email);
        return usuario;
    }
}

// tests/usuario.test.js
describe('UsuarioService', () => {
    let usuarioService;
    let mockRepository;
    let mockEmailService;

    beforeEach(() => {
        mockRepository = {
            crear: jest.fn()
        };
        mockEmailService = {
            enviarBienvenida: jest.fn()
        };
        usuarioService = new UsuarioService(mockRepository, mockEmailService);
    });

    test('debe crear usuario y enviar email', async () => {
        const datosUsuario = { 
            email: 'test@test.com', 
            nombre: 'Test' 
        };
        const usuarioCreado = { 
            ...datosUsuario, 
            id: 1 
        };

        mockRepository.crear.mockResolvedValue(usuarioCreado);
        mockEmailService.enviarBienvenida.mockResolvedValue(true);

        const resultado = await usuarioService.crearUsuario(datosUsuario);

        expect(mockRepository.crear).toHaveBeenCalledWith(datosUsuario);
        expect(mockEmailService.enviarBienvenida).toHaveBeenCalledWith(datosUsuario.email);
        expect(resultado).toEqual(usuarioCreado);
    });
});
```

## 2. Integration Testing

### 2.1 Testing de API con Supertest
```javascript
// tests/api.test.js
const request = require('supertest');
const app = require('../app');
const mongoose = require('mongoose');

describe('API Tests', () => {
    beforeAll(async () => {
        await mongoose.connect(process.env.MONGO_URI_TEST);
    });

    afterAll(async () => {
        await mongoose.connection.dropDatabase();
        await mongoose.connection.close();
    });

    describe('POST /api/usuarios', () => {
        test('debe crear un nuevo usuario', async () => {
            const datosUsuario = {
                nombre: 'Test User',
                email: 'test@test.com',
                password: 'password123'
            };

            const response = await request(app)
                .post('/api/usuarios')
                .send(datosUsuario)
                .expect(201);

            expect(response.body).toHaveProperty('id');
            expect(response.body.email).toBe(datosUsuario.email);
            expect(response.body).not.toHaveProperty('password');
        });

        test('debe retornar error si falta email', async () => {
            const datosUsuario = {
                nombre: 'Test User',
                password: 'password123'
            };

            const response = await request(app)
                .post('/api/usuarios')
                .send(datosUsuario)
                .expect(400);

            expect(response.body).toHaveProperty('error');
            expect(response.body.error).toContain('email');
        });
    });
});
```

### 2.2 Testing de Base de Datos
```javascript
// tests/db.test.js
describe('Database Tests', () => {
    let connection;
    let db;

    beforeAll(async () => {
        connection = await MongoClient.connect(globalThis.__MONGO_URI__, {
            useNewUrlParser: true,
            useUnifiedTopology: true,
        });
        db = await connection.db(globalThis.__MONGO_DB_NAME__);
    });

    afterAll(async () => {
        await connection.close();
    });

    beforeEach(async () => {
        await db.collection('usuarios').deleteMany({});
    });

    test('debe insertar un nuevo usuario', async () => {
        const usuarios = db.collection('usuarios');

        const mockUsuario = { nombre: 'Test', email: 'test@test.com' };
        await usuarios.insertOne(mockUsuario);

        const insertedUser = await usuarios.findOne({ email: 'test@test.com' });
        expect(insertedUser).toEqual(expect.objectContaining(mockUsuario));
    });
});
```

## 3. End-to-End Testing

### 3.1 Testing con Cypress
```javascript
// cypress/integration/auth.spec.js
describe('Autenticación', () => {
    beforeEach(() => {
        cy.visit('/login');
    });

    it('debe permitir login con credenciales válidas', () => {
        cy.get('[data-cy=email-input]')
            .type('usuario@test.com');
        
        cy.get('[data-cy=password-input]')
            .type('password123');
        
        cy.get('[data-cy=login-button]')
            .click();

        cy.url().should('include', '/dashboard');
        cy.get('[data-cy=welcome-message]')
            .should('contain', 'Bienvenido');
    });

    it('debe mostrar error con credenciales inválidas', () => {
        cy.get('[data-cy=email-input]')
            .type('usuario@test.com');
        
        cy.get('[data-cy=password-input]')
            .type('wrongpassword');
        
        cy.get('[data-cy=login-button]')
            .click();

        cy.get('[data-cy=error-message]')
            .should('be.visible')
            .and('contain', 'Credenciales inválidas');
    });
});
```

## 4. CI/CD con GitHub Actions

### 4.1 Configuración de Pipeline
```yaml
# .github/workflows/node.yml
name: Node.js CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [14.x, 16.x]
        mongodb-version: ['4.4']

    steps:
    - uses: actions/checkout@v2

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v2
      with:
        node-version: ${{ matrix.node-version }}

    - name: Start MongoDB
      uses: supercharge/mongodb-github-action@1.7.0
      with:
        mongodb-version: ${{ matrix.mongodb-version }}

    - name: Install dependencies
      run: npm ci

    - name: Run linter
      run: npm run lint

    - name: Run tests
      run: npm test

    - name: Upload coverage
      uses: codecov/codecov-action@v2
```

## Proyecto Práctico: API con Cobertura Completa

Este proyecto demuestra la implementación de tests en una API REST:

```javascript
// src/services/post.service.js
class PostService {
    constructor(postRepository, userService) {
        this.postRepository = postRepository;
        this.userService = userService;
    }

    async crearPost(userId, postData) {
        const user = await this.userService.getById(userId);
        if (!user) {
            throw new Error('Usuario no encontrado');
        }

        return this.postRepository.create({
            ...postData,
            userId,
            createdAt: new Date()
        });
    }
}

// tests/integration/post.test.js
describe('Post API', () => {
    let token;
    let userId;

    beforeAll(async () => {
        // Setup: crear usuario y obtener token
        const response = await request(app)
            .post('/api/auth/login')
            .send({
                email: 'test@test.com',
                password: 'password123'
            });

        token = response.body.token;
        userId = response.body.userId;
    });

    describe('POST /api/posts', () => {
        test('debe crear un nuevo post', async () => {
            const postData = {
                title: 'Test Post',
                content: 'Test Content'
            };

            const response = await request(app)
                .post('/api/posts')
                .set('Authorization', `Bearer ${token}`)
                .send(postData)
                .expect(201);

            expect(response.body).toMatchObject({
                title: postData.title,
                content: postData.content,
                userId
            });
        });
    });
});

// tests/unit/post.service.test.js
describe('PostService', () => {
    let postService;
    let mockPostRepository;
    let mockUserService;

    beforeEach(() => {
        mockPostRepository = {
            create: jest.fn()
        };
        mockUserService = {
            getById: jest.fn()
        };
        postService = new PostService(mockPostRepository, mockUserService);
    });

    describe('crearPost', () => {
        test('debe crear post para usuario existente', async () => {
            const userId = '123';
            const postData = {
                title: 'Test',
                content: 'Content'
            };
            const user = { id: userId, name: 'Test User' };
            
            mockUserService.getById.mockResolvedValue(user);
            mockPostRepository.create.mockResolvedValue({
                ...postData,
                userId,
                id: '456'
            });

            const result = await postService.crearPost(userId, postData);

            expect(mockUserService.getById).toHaveBeenCalledWith(userId);
            expect(mockPostRepository.create).toHaveBeenCalledWith(
                expect.objectContaining({
                    ...postData,
                    userId
                })
            );
            expect(result).toHaveProperty('id');
        });

        test('debe lanzar error si usuario no existe', async () => {
            mockUserService.getById.mockResolvedValue(null);

            await expect(
                postService.crearPost('123', {})
            ).rejects.toThrow('Usuario no encontrado');
        });
    });
});
```

Este proyecto demuestra:
- Tests unitarios con Jest
- Tests de integración con Supertest
- Mocking y spies
- Cobertura de código
- CI/CD con GitHub Actions
- Testing de bases de datos
- Testing de APIs
- Best practices en testing
