# FWD-Exercises
Crear API Rest NodeJS
user
Crear un paso a paso para un construir un Rest API con NodeJS Express, MySQL, Sequelize nivel intermedio/Avanzado que maneje Autenticacion JWT en sus consultas
ChatGPT
Aquí tienes una guía paso a paso para crear una REST API intermedia/avanzada en Node.js usando Express, MySQL, Sequelize y JWT para la autenticación:

---

### Paso 1: Configuración del Proyecto

1. **Inicializa el proyecto**:
   ```bash
   mkdir my-api
   cd my-api
   npm init -y
   ```

2. **Instala las dependencias**:
   ```bash
   npm install express sequelize mysql2 jsonwebtoken bcryptjs dotenv
   ```

3. **Configura el archivo `.env`** para almacenar tus variables de entorno:
   ```plaintext
   DB_HOST=localhost
   DB_USER=tu_usuario
   DB_PASSWORD=tu_contraseña
   DB_NAME=nombre_bd
   JWT_SECRET=tu_clave_secreta
   JWT_EXPIRES_IN=1d
   ```

### Paso 2: Configura Sequelize y Conecta a la Base de Datos

1. **Crea la configuración de Sequelize** en un archivo `config/database.js`:
   ```javascript
   require('dotenv').config();
   const { Sequelize } = require('sequelize');

   const sequelize = new Sequelize(process.env.DB_NAME, process.env.DB_USER, process.env.DB_PASSWORD, {
     host: process.env.DB_HOST,
     dialect: 'mysql',
   });

   module.exports = sequelize;
   ```

2. **Conéctate a la base de datos** en `index.js`:
   ```javascript
   const express = require('express');
   const sequelize = require('./config/database');
   const app = express();

   sequelize.authenticate()
     .then(() => console.log('Conexión a la base de datos exitosa'))
     .catch((err) => console.log('Error al conectar a la base de datos', err));

   app.listen(3000, () => console.log('Servidor corriendo en puerto 3000'));
   ```

### Paso 3: Define los Modelos

1. **Crea el modelo de Usuario** en `models/User.js`:
   ```javascript
   const { DataTypes } = require('sequelize');
   const sequelize = require('../config/database');

   const User = sequelize.define('User', {
     id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
     username: { type: DataTypes.STRING, allowNull: false, unique: true },
     email: { type: DataTypes.STRING, allowNull: false, unique: true },
     password: { type: DataTypes.STRING, allowNull: false },
   });

   module.exports = User;
   ```

2. **Sincroniza los modelos con la base de datos** (opcional en desarrollo):
   ```javascript
   sequelize.sync({ force: true });
   ```

### Paso 4: Implementa la Autenticación JWT

1. **Crea el controlador de autenticación** en `controllers/authController.js`:
   ```javascript
   const jwt = require('jsonwebtoken');
   const bcrypt = require('bcryptjs');
   const User = require('../models/User');

   exports.register = async (req, res) => {
     try {
       const { username, email, password } = req.body;
       const hashedPassword = await bcrypt.hash(password, 10);
       const user = await User.create({ username, email, password: hashedPassword });
       res.json({ user });
     } catch (error) {
       res.status(500).json({ error: 'Error al registrar usuario' });
     }
   };

   exports.login = async (req, res) => {
     try {
       const { email, password } = req.body;
       const user = await User.findOne({ where: { email } });
       if (!user || !(await bcrypt.compare(password, user.password))) {
         return res.status(401).json({ error: 'Credenciales incorrectas' });
       }
       const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRES_IN });
       res.json({ token });
     } catch (error) {
       res.status(500).json({ error: 'Error al iniciar sesión' });
     }
   };
   ```

2. **Crea la ruta de autenticación** en `routes/auth.js`:
   ```javascript
   const express = require('express');
   const authController = require('../controllers/authController');
   const router = express.Router();

   router.post('/register', authController.register);
   router.post('/login', authController.login);

   module.exports = router;
   ```

3. **Agrega las rutas al servidor** en `index.js`:
   ```javascript
   const authRoutes = require('./routes/auth');
   app.use(express.json());
   app.use('/auth', authRoutes);
   ```

### Paso 5: Implementa el Middleware de Autenticación

1. **Crea el middleware de autenticación JWT** en `middlewares/authMiddleware.js`:
   ```javascript
   const jwt = require('jsonwebtoken');

   const authenticate = (req, res, next) => {
     const token = req.headers.authorization?.split(' ')[1];
     if (!token) return res.status(401).json({ error: 'Token no proporcionado' });

     try {
       const decoded = jwt.verify(token, process.env.JWT_SECRET);
       req.userId = decoded.id;
       next();
     } catch (error) {
       res.status(401).json({ error: 'Token inválido' });
     }
   };

   module.exports = authenticate;
   ```

### Paso 6: Crea Rutas Protegidas

1. **Crea un controlador protegido de ejemplo** en `controllers/profileController.js`:
   ```javascript
   const User = require('../models/User');

   exports.profile = async (req, res) => {
     try {
       const user = await User.findByPk(req.userId);
       res.json({ user });
     } catch (error) {
       res.status(500).json({ error: 'Error al obtener perfil' });
     }
   };
   ```

2. **Crea las rutas protegidas** en `routes/profile.js`:
   ```javascript
   const express = require('express');
   const profileController = require('../controllers/profileController');
   const authenticate = require('../middlewares/authMiddleware');
   const router = express.Router();

   router.get('/', authenticate, profileController.profile);

   module.exports = router;
   ```

3. **Integra las rutas protegidas** en `index.js`:
   ```javascript
   const profileRoutes = require('./routes/profile');
   app.use('/profile', profileRoutes);
   ```

### Paso 7: Prueba la API

1. **Registro de usuario**:
   - Ruta: `POST /auth/register`
   - Body:
     ```json
     {
       "username": "user1",
       "email": "user1@example.com",
       "password": "password123"
     }
     ```

2. **Inicio de sesión**:
   - Ruta: `POST /auth/login`
   - Body:
     ```json
     {
       "email": "user1@example.com",
       "password": "password123"
     }
     ```

3. **Acceso a la ruta protegida**:
   - Ruta: `GET /profile`
   - Header: `Authorization: Bearer <token_obtenido_en_login>`

---

Este flujo proporciona una API REST en Node.js usando JWT para autenticación, Sequelize para la ORM con MySQL y rutas protegidas. ¡Con esta base puedes expandir fácilmente la API para añadir funcionalidades avanzadas!
user
Segun lo anterior puedes implementar el mismo ejercicio utilizando sequelize-cli
ChatGPT
Para crear una REST API en Node.js con Express, MySQL, Sequelize y autenticación JWT usando `sequelize-cli`, los pasos serán similares al ejercicio anterior, pero utilizaremos `sequelize-cli` para automatizar la creación de archivos de configuración, modelos y migraciones.

### Paso 1: Inicialización del Proyecto

1. **Crea el proyecto e instala dependencias**:
   ```bash
   mkdir my-api
   cd my-api
   npm init -y
   npm install express sequelize mysql2 jsonwebtoken bcryptjs dotenv
   npm install --save-dev sequelize-cli
   ```

2. **Inicializa Sequelize CLI**:
   ```bash
   npx sequelize-cli init
   ```

   Esto creará la siguiente estructura de carpetas:
   ```
   ├── config
   │   └── config.json
   ├── models
   │   └── index.js
   ├── migrations
   ├── seeders
   ```

3. **Configura la base de datos** en `config/config.json` para conectar a MySQL:
   ```json
   {
     "development": {
       "username": "tu_usuario",
       "password": "tu_contraseña",
       "database": "nombre_bd",
       "host": "127.0.0.1",
       "dialect": "mysql"
     }
   }
   ```

### Paso 2: Crear el Modelo y Migración de Usuario

1. **Crea el modelo de `User`** con `sequelize-cli`:
   ```bash
   npx sequelize-cli model:generate --name User --attributes username:string,email:string,password:string
   ```

   Esto generará un archivo de modelo en `models/user.js` y una migración en `migrations/<timestamp>-create-user.js`.

2. **Modifica el archivo de migración** en `migrations/<timestamp>-create-user.js` para definir los campos de `User`:
   ```javascript
   'use strict';
   module.exports = {
     up: async (queryInterface, Sequelize) => {
       await queryInterface.createTable('Users', {
         id: {
           allowNull: false,
           autoIncrement: true,
           primaryKey: true,
           type: Sequelize.INTEGER
         },
         username: {
           type: Sequelize.STRING,
           allowNull: false,
           unique: true
         },
         email: {
           type: Sequelize.STRING,
           allowNull: false,
           unique: true
         },
         password: {
           type: Sequelize.STRING,
           allowNull: false
         },
         createdAt: {
           allowNull: false,
           type: Sequelize.DATE
         },
         updatedAt: {
           allowNull: false,
           type: Sequelize.DATE
         }
       });
     },
     down: async (queryInterface, Sequelize) => {
       await queryInterface.dropTable('Users');
     }
   };
   ```

3. **Ejecuta la migración** para crear la tabla `Users`:
   ```bash
   npx sequelize-cli db:migrate
   ```

### Paso 3: Configuración de JWT y Autenticación

1. **Configura las variables de entorno** en un archivo `.env`:
   ```plaintext
   JWT_SECRET=tu_clave_secreta
   JWT_EXPIRES_IN=1d
   ```

2. **Crea el controlador de autenticación** en `controllers/authController.js`:
   ```javascript
   const jwt = require('jsonwebtoken');
   const bcrypt = require('bcryptjs');
   const { User } = require('../models');

   exports.register = async (req, res) => {
     try {
       const { username, email, password } = req.body;
       const hashedPassword = await bcrypt.hash(password, 10);
       const user = await User.create({ username, email, password: hashedPassword });
       res.json({ user });
     } catch (error) {
       res.status(500).json({ error: 'Error al registrar usuario' });
     }
   };

   exports.login = async (req, res) => {
     try {
       const { email, password } = req.body;
       const user = await User.findOne({ where: { email } });
       if (!user || !(await bcrypt.compare(password, user.password))) {
         return res.status(401).json({ error: 'Credenciales incorrectas' });
       }
       const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRES_IN });
       res.json({ token });
     } catch (error) {
       res.status(500).json({ error: 'Error al iniciar sesión' });
     }
   };
   ```

3. **Crea el middleware de autenticación JWT** en `middlewares/authMiddleware.js`:
   ```javascript
   const jwt = require('jsonwebtoken');

   const authenticate = (req, res, next) => {
     const token = req.headers.authorization?.split(' ')[1];
     if (!token) return res.status(401).json({ error: 'Token no proporcionado' });

     try {
       const decoded = jwt.verify(token, process.env.JWT_SECRET);
       req.userId = decoded.id;
       next();
     } catch (error) {
       res.status(401).json({ error: 'Token inválido' });
     }
   };

   module.exports = authenticate;
   ```

### Paso 4: Configuración de Rutas

1. **Crea las rutas de autenticación** en `routes/auth.js`:
   ```javascript
   const express = require('express');
   const authController = require('../controllers/authController');
   const router = express.Router();

   router.post('/register', authController.register);
   router.post('/login', authController.login);

   module.exports = router;
   ```

2. **Crea las rutas protegidas** en `routes/profile.js` y un controlador de perfil:
   ```javascript
   const express = require('express');
   const authenticate = require('../middlewares/authMiddleware');
   const { User } = require('../models');
   const router = express.Router();

   router.get('/', authenticate, async (req, res) => {
     try {
       const user = await User.findByPk(req.userId);
       res.json({ user });
     } catch (error) {
       res.status(500).json({ error: 'Error al obtener perfil' });
     }
   });

   module.exports = router;
   ```

### Paso 5: Configuración del Servidor

1. **Configura el servidor** en `index.js`:
   ```javascript
   const express = require('express');
   const authRoutes = require('./routes/auth');
   const profileRoutes = require('./routes/profile');
   const app = express();

   app.use(express.json());
   app.use('/auth', authRoutes);
   app.use('/profile', profileRoutes);

   const PORT = process.env.PORT || 3000;
   app.listen(PORT, () => console.log(`Servidor corriendo en el puerto ${PORT}`));
   ```

### Paso 6: Ejecuta y Prueba la API

1. **Ejecuta el servidor**:
   ```bash
   node index.js
   ```

2. **Prueba los endpoints**:
   - **Registro de usuario**:
     ```plaintext
     POST /auth/register
     Body: { "username": "user1", "email": "user1@example.com", "password": "password123" }
     ```
   - **Inicio de sesión**:
     ```plaintext
     POST /auth/login
     Body: { "email": "user1@example.com", "password": "password123" }
     ```
   - **Ruta protegida (perfil)**:
     ```plaintext
     GET /profile
     Header: Authorization: Bearer <token_obtenido_en_login>
     ```

Este flujo permite usar `sequelize-cli` para gestionar la estructura de la base de datos y automatizar el manejo de modelos y migraciones. ¡Con estos pasos, tendrás una API robusta con autenticación JWT lista para su uso!
user
Siguiendo la misma linea del ejercicio anterior puedes dirigir el API a que focalice un sistema de reserva de vuelos?
ChatGPT
Para adaptar el ejercicio anterior a un sistema de reserva de vuelos, vamos a ajustar los modelos, migraciones y controladores para gestionar **Usuarios**, **Vuelos** y **Reservas**. Este sistema permitirá a los usuarios registrarse, iniciar sesión, ver vuelos disponibles y hacer reservas. Usaremos `sequelize-cli` para la generación de los modelos y migraciones.

### Paso 1: Inicialización del Proyecto

Comienza con los mismos pasos para configurar el proyecto, asegurándote de tener `express`, `sequelize`, `mysql2`, `jsonwebtoken`, `bcryptjs`, `dotenv`, y `sequelize-cli` instalados. Para los detalles, puedes consultar el primer paso de la respuesta anterior.

### Paso 2: Crear Modelos y Migraciones

1. **Modelo de Usuario**:
   Este modelo sigue siendo el mismo que en el ejercicio anterior. Si ya tienes un modelo de `User`, puedes saltarte este paso.

2. **Modelo de Vuelo**:
   Genera un modelo de vuelo (`Flight`) con `sequelize-cli`:
   ```bash
   npx sequelize-cli model:generate --name Flight --attributes origin:string,destination:string,departureTime:date,arrivalTime:date,price:decimal
   ```

   La estructura del modelo `Flight` en `models/flight.js` debería tener algo como esto:
   ```javascript
   'use strict';
   const { Model } = require('sequelize');
   module.exports = (sequelize, DataTypes) => {
     class Flight extends Model {
       static associate(models) {
         Flight.hasMany(models.Reservation, { foreignKey: 'flightId' });
       }
     }
     Flight.init({
       origin: DataTypes.STRING,
       destination: DataTypes.STRING,
       departureTime: DataTypes.DATE,
       arrivalTime: DataTypes.DATE,
       price: DataTypes.DECIMAL
     }, {
       sequelize,
       modelName: 'Flight',
     });
     return Flight;
   };
   ```

3. **Modelo de Reserva**:
   Genera un modelo de reserva (`Reservation`) que relaciona usuarios y vuelos:
   ```bash
   npx sequelize-cli model:generate --name Reservation --attributes userId:integer,flightId:integer,status:string
   ```

   La estructura del modelo `Reservation` en `models/reservation.js` debería verse así:
   ```javascript
   'use strict';
   const { Model } = require('sequelize');
   module.exports = (sequelize, DataTypes) => {
     class Reservation extends Model {
       static associate(models) {
         Reservation.belongsTo(models.User, { foreignKey: 'userId' });
         Reservation.belongsTo(models.Flight, { foreignKey: 'flightId' });
       }
     }
     Reservation.init({
       userId: DataTypes.INTEGER,
       flightId: DataTypes.INTEGER,
       status: DataTypes.STRING
     }, {
       sequelize,
       modelName: 'Reservation',
     });
     return Reservation;
   };
   ```

4. **Ejecuta las migraciones** para crear las tablas correspondientes en la base de datos:
   ```bash
   npx sequelize-cli db:migrate
   ```

### Paso 3: Controladores

1. **Controlador de Vuelos** en `controllers/flightController.js`:
   Este controlador permitirá listar vuelos y buscar vuelos por origen, destino o rango de fechas.

   ```javascript
   const { Flight } = require('../models');

   exports.getAllFlights = async (req, res) => {
     try {
       const flights = await Flight.findAll();
       res.json(flights);
     } catch (error) {
       res.status(500).json({ error: 'Error al obtener vuelos' });
     }
   };

   exports.getFlightById = async (req, res) => {
     try {
       const flight = await Flight.findByPk(req.params.id);
       if (!flight) return res.status(404).json({ error: 'Vuelo no encontrado' });
       res.json(flight);
     } catch (error) {
       res.status(500).json({ error: 'Error al obtener el vuelo' });
     }
   };
   ```

2. **Controlador de Reservas** en `controllers/reservationController.js`:
   Este controlador permitirá crear y gestionar reservas de vuelos.

   ```javascript
   const { Reservation, Flight } = require('../models');

   exports.createReservation = async (req, res) => {
     try {
       const { flightId } = req.body;
       const flight = await Flight.findByPk(flightId);

       if (!flight) return res.status(404).json({ error: 'Vuelo no encontrado' });

       const reservation = await Reservation.create({
         userId: req.userId,
         flightId,
         status: 'confirmado'
       });

       res.json({ message: 'Reserva creada con éxito', reservation });
     } catch (error) {
       res.status(500).json({ error: 'Error al crear la reserva' });
     }
   };

   exports.getReservationsByUser = async (req, res) => {
     try {
       const reservations = await Reservation.findAll({
         where: { userId: req.userId },
         include: [{ model: Flight }]
       });
       res.json(reservations);
     } catch (error) {
       res.status(500).json({ error: 'Error al obtener reservas' });
     }
   };
   ```

### Paso 4: Rutas

1. **Rutas de Vuelos** en `routes/flight.js`:
   ```javascript
   const express = require('express');
   const flightController = require('../controllers/flightController');
   const router = express.Router();

   router.get('/', flightController.getAllFlights);
   router.get('/:id', flightController.getFlightById);

   module.exports = router;
   ```

2. **Rutas de Reservas** en `routes/reservation.js`:
   ```javascript
   const express = require('express');
   const reservationController = require('../controllers/reservationController');
   const authenticate = require('../middlewares/authMiddleware');
   const router = express.Router();

   router.post('/', authenticate, reservationController.createReservation);
   router.get('/', authenticate, reservationController.getReservationsByUser);

   module.exports = router;
   ```

### Paso 5: Configuración del Servidor

En el archivo principal `index.js`, agrega las rutas de vuelos y reservas junto con las rutas de autenticación:
```javascript
const express = require('express');
const authRoutes = require('./routes/auth');
const flightRoutes = require('./routes/flight');
const reservationRoutes = require('./routes/reservation');
const app = express();

app.use(express.json());
app.use('/auth', authRoutes);
app.use('/flights', flightRoutes);
app.use('/reservations', reservationRoutes);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Servidor corriendo en el puerto ${PORT}`));
```

### Paso 6: Pruebas de la API

1. **Vuelos**:
   - Obtener todos los vuelos:
     ```plaintext
     GET /flights
     ```
   - Obtener un vuelo específico:
     ```plaintext
     GET /flights/:id
     ```

2. **Reservas**:
   - Crear una reserva (autenticado):
     ```plaintext
     POST /reservations
     Header: Authorization: Bearer <token>
     Body: { "flightId": 1 }
     ```
   - Obtener todas las reservas del usuario (autenticado):
     ```plaintext
     GET /reservations
     Header: Authorization: Bearer <token>
     ```

Este flujo permite crear y gestionar reservas de vuelos. La estructura modular también facilita futuras expansiones o mejoras en el sistema, como cancelación de reservas, cambios de vuelo o sistemas de pago.
