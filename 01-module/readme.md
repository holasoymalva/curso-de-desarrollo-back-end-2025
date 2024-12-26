# Módulo 1: Fundamentos de Node.js y Arquitectura Backend

## 1. Introducción a Node.js y su Arquitectura

### 1.1 ¿Qué es Node.js?

Node.js es un entorno de ejecución de JavaScript del lado del servidor construido sobre el motor V8 de Chrome. Sus características principales son:

- **Runtime de JavaScript**: Permite ejecutar código JavaScript fuera del navegador
- **Basado en V8**: Utiliza el motor V8 de Google Chrome para ejecutar código JavaScript
- **Orientado a Eventos**: Arquitectura basada en un bucle de eventos (Event Loop)
- **No Bloqueante**: Operaciones de I/O asíncronas por defecto
- **Multiplataforma**: Funciona en Windows, Mac, Linux, etc.

Ventajas principales:
- Alta velocidad de ejecución
- Gran ecosistema de paquetes (NPM)
- Excelente para aplicaciones en tiempo real
- Mismo lenguaje en frontend y backend

### 1.2 Arquitectura de Node.js

#### Componentes Principales:
1. **V8 Engine**:
   - Motor JavaScript de Google
   - Compila JavaScript a código máquina
   - Gestiona la memoria (Garbage Collection)

2. **libuv**:
   - Biblioteca de I/O asíncrono
   - Gestiona el Event Loop
   - Maneja operaciones del sistema operativo

3. **Event Loop**:
```javascript
// Ejemplo práctico del Event Loop
console.log('1 - Tarea síncrona');

setTimeout(() => {
    console.log('2 - Timer callback (macrotarea)');
}, 0);

Promise.resolve().then(() => {
    console.log('3 - Promise (microtarea)');
});

process.nextTick(() => {
    console.log('4 - NextTick (microtarea prioritaria)');
});

console.log('5 - Tarea síncrona');

// Salida:
// 1 - Tarea síncrona
// 5 - Tarea síncrona
// 4 - NextTick (microtarea prioritaria)
// 3 - Promise (microtarea)
// 2 - Timer callback (macrotarea)
```

#### Fases del Event Loop:
1. **timers**: Ejecuta callbacks de setTimeout/setInterval
2. **pending callbacks**: Ejecuta callbacks de I/O pendientes
3. **idle, prepare**: Uso interno de Node.js
4. **poll**: Recupera nuevos eventos de I/O
5. **check**: Ejecuta callbacks de setImmediate
6. **close callbacks**: Ejecuta callbacks de cierre

## 2. Sistema de Módulos y NPM

### 2.1 Módulos en Node.js

#### Tipos de Módulos:

1. **Módulos Core (Built-in)**:
```javascript
// Módulos incorporados
const fs = require('fs');
const path = require('path');
const http = require('http');
```

2. **Módulos Locales**:
```javascript
// matematica.js
class Calculadora {
    suma(a, b) {
        return a + b;
    }
    
    resta(a, b) {
        return a - b;
    }
    
    multiplica(a, b) {
        return a * b;
    }
    
    divide(a, b) {
        if (b === 0) throw new Error('No se puede dividir por cero');
        return a / b;
    }
}

module.exports = Calculadora;

// index.js
const Calculadora = require('./matematica');
const calc = new Calculadora();
console.log(calc.suma(5, 3));     // 8
console.log(calc.multiplica(4, 2)) // 8
```

3. **Módulos de Terceros**:
```javascript
// Instalación: npm install axios
const axios = require('axios');

async function obtenerDatos() {
    try {
        const response = await axios.get('https://api.ejemplo.com/datos');
        console.log(response.data);
    } catch (error) {
        console.error('Error:', error.message);
    }
}
```

### 2.2 NPM (Node Package Manager)

#### Comandos Esenciales:
```bash
# Iniciar nuevo proyecto
npm init

# Instalación de dependencias
npm install express         # Dependencia de producción
npm install --save-dev jest # Dependencia de desarrollo

# Scripts personalizados
```

#### package.json explicado:
```json
{
  "name": "mi-proyecto",
  "version": "1.0.0",
  "description": "Descripción del proyecto",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.17.1"
  },
  "devDependencies": {
    "nodemon": "^2.0.7",
    "jest": "^27.0.6"
  }
}
```

## 3. Event Loop y Programación Asíncrona

### 3.1 Callbacks y Promesas

#### Callbacks:
```javascript
// Ejemplo de callback hell
fs.readFile('archivo1.txt', 'utf8', (error1, data1) => {
    if (error1) {
        console.error('Error:', error1);
        return;
    }
    fs.readFile('archivo2.txt', 'utf8', (error2, data2) => {
        if (error2) {
            console.error('Error:', error2);
            return;
        }
        fs.writeFile('archivo3.txt', data1 + data2, (error3) => {
            if (error3) {
                console.error('Error:', error3);
                return;
            }
            console.log('Archivos procesados exitosamente');
        });
    });
});
```

#### Promesas:
```javascript
// Mismo ejemplo con Promesas
const fs = require('fs').promises;

fs.readFile('archivo1.txt', 'utf8')
    .then(data1 => {
        return fs.readFile('archivo2.txt', 'utf8')
            .then(data2 => {
                return fs.writeFile('archivo3.txt', data1 + data2);
            });
    })
    .then(() => {
        console.log('Archivos procesados exitosamente');
    })
    .catch(error => {
        console.error('Error:', error);
    });
```

#### Async/Await:
```javascript
// Mismo ejemplo con async/await
async function procesarArchivos() {
    try {
        const data1 = await fs.readFile('archivo1.txt', 'utf8');
        const data2 = await fs.readFile('archivo2.txt', 'utf8');
        await fs.writeFile('archivo3.txt', data1 + data2);
        console.log('Archivos procesados exitosamente');
    } catch (error) {
        console.error('Error:', error);
    }
}

procesarArchivos();
```

### 3.2 Manejo de Errores Asíncronos

#### Patrones de Manejo de Errores:
```javascript
// 1. Try-Catch con Async/Await
async function manejoErrores() {
    try {
        await operacionAsincrona();
    } catch (error) {
        if (error instanceof TypeError) {
            console.error('Error de tipo:', error.message);
        } else if (error instanceof NetworkError) {
            console.error('Error de red:', error.message);
        } else {
            console.error('Error desconocido:', error);
        }
    } finally {
        console.log('Limpieza final');
    }
}

// 2. Errores personalizados
class DatabaseError extends Error {
    constructor(message) {
        super(message);
        this.name = 'DatabaseError';
    }
}

async function conectarDB() {
    throw new DatabaseError('Error de conexión a la base de datos');
}
```

## 4. Streams y Buffers

### 4.1 Tipos de Streams

#### Readable Streams:
```javascript
const fs = require('fs');

const readStream = fs.createReadStream('archivo-grande.txt', {
    encoding: 'utf8',
    highWaterMark: 64 * 1024 // 64KB chunks
});

readStream.on('data', (chunk) => {
    console.log('Recibido:', chunk.length, 'bytes');
});

readStream.on('end', () => {
    console.log('Lectura finalizada');
});

readStream.on('error', (error) => {
    console.error('Error:', error.message);
});
```

#### Writable Streams:
```javascript
const writeStream = fs.createWriteStream('salida.txt');

writeStream.write('Línea 1\n');
writeStream.write('Línea 2\n');
writeStream.end('Última línea');

writeStream.on('finish', () => {
    console.log('Escritura completada');
});
```

#### Transform Streams:
```javascript
const { Transform } = require('stream');

class ConvertidorMayusculas extends Transform {
    _transform(chunk, encoding, callback) {
        const upperChunk = chunk.toString().toUpperCase();
        callback(null, upperChunk);
    }
}

const convertidor = new ConvertidorMayusculas();
readStream
    .pipe(convertidor)
    .pipe(writeStream);
```

### 4.2 Buffers

#### Operaciones con Buffers:
```javascript
// Crear buffers
const buf1 = Buffer.from('Hola Mundo');
const buf2 = Buffer.alloc(10); // Buffer vacío de 10 bytes

// Operaciones comunes
console.log(buf1.toString());        // Hola Mundo
console.log(buf1.length);            // 10
console.log(buf1.toJSON());          // { type: 'Buffer', data: [ ... ] }

// Concatenar buffers
const buf3 = Buffer.concat([buf1, buf2]);

// Comparar buffers
console.log(buf1.equals(buf2));      // false

// Copiar buffers
buf1.copy(buf2);
```

## Proyecto Práctico: Sistema de Procesamiento de Logs

Este proyecto integra todos los conceptos aprendidos:

```javascript
const fs = require('fs');
const { Transform } = require('stream');
const path = require('path');

class ProcesadorLogs extends Transform {
    constructor(options = {}) {
        super(options);
        this.errores = fs.createWriteStream('errores.log');
        this.warnings = fs.createWriteStream('warnings.log');
        this.info = fs.createWriteStream('info.log');
    }

    _transform(chunk, encoding, callback) {
        const linea = chunk.toString();
        
        try {
            if (linea.includes('ERROR')) {
                this.errores.write(linea + '\n');
            } else if (linea.includes('WARNING')) {
                this.warnings.write(linea + '\n');
            } else if (linea.includes('INFO')) {
                this.info.write(linea + '\n');
            }
            
            // Pasar la línea sin modificar
            callback(null, chunk);
        } catch (error) {
            callback(error);
        }
    }

    _flush(callback) {
        this.errores.end();
        this.warnings.end();
        this.info.end();
        callback();
    }
}

async function procesarArchivosLogs(directorio) {
    try {
        const archivos = await fs.promises.readdir(directorio);
        const logsArchivos = archivos.filter(archivo => archivo.endsWith('.log'));
        
        const procesador = new ProcesadorLogs();
        
        for (const archivo of logsArchivos) {
            const rutaArchivo = path.join(directorio, archivo);
            console.log(`Procesando: ${rutaArchivo}`);
            
            const readStream = fs.createReadStream(rutaArchivo);
            
            await new Promise((resolve, reject) => {
                readStream
                    .pipe(procesador)
                    .on('finish', resolve)
                    .on('error', reject);
            });
        }
        
        console.log('Procesamiento completado');
    } catch (error) {
        console.error('Error durante el procesamiento:', error);
    }
}

// Uso
procesarArchivosLogs('./logs');
```

Este proyecto demuestra:
- Uso de streams para manejo eficiente de archivos
- Transformación de datos en tiempo real
- Manejo de errores asíncronos
- Programación orientada a eventos
- Módulos y clases de Node.js
