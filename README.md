# RestaurantApp
This is a restaurant reservation Android app

Ακολουθούν οι οδηγίες εγκατάστασης και η περιγραφή λειτουργικότητας για την εφαρμογή:

Οδηγίες εγκατάστασης:

Backend:

Βεβαιωθείτε ότι έχετε εγκατεστημένο το Node.js και την MySQL.
Δημιουργήστε ένα αρχείο .env στο root του backend project και προσθέστε τη μεταβλητή JWT_SECRET με μια μυστική τιμή (π.χ., JWT_SECRET=supersecretkey).    
Εγκαταστήστε τις απαραίτητες dependencies εκτελώντας npm install στο τερματικό.
Ρυθμίστε τη σύνδεση με τη βάση δεδομένων MySQL στο backend.js με τα στοιχεία της δικής σας εγκατάστασης (host, user, password, database).    
Δημιουργήστε τη βάση δεδομένων με το όνομα restaurantprogram (ή όποιο άλλο έχετε ορίσει στο backend.js) και τα απαραίτητα tables (users, restaurants, reservations).
Εκκινήστε τον backend server εκτελώντας npm start ή node backend.js. Ο server θα ξεκινήσει να ακούει στην πόρτα 5000 ή την πόρτα που θα ορίσετε στο .env αρχείο σας.    
Frontend:

Βεβαιωθείτε ότι έχετε εγκατεστημένο το Node.js και το npm.
Μεταβείτε στο φάκελο του frontend project στο τερματικό.
Εγκαταστήστε τις απαραίτητες dependencies εκτελώντας npm install.
Εκκινήστε την εφαρμογή εκτελώντας npm start. Η εφαρμογή θα ανοίξει στο browser σας (συνήθως στην διεύθυνση http://localhost:3000).
Περιγραφή λειτουργικότητας:

Η εφαρμογή παρέχει τις εξής λειτουργίες:

Εγγραφή χρήστη: Οι χρήστες μπορούν να δημιουργήσουν νέους λογαριασμούς παρέχοντας όνομα, email και κωδικό πρόσβασης.    
Σύνδεση χρήστη: Οι εγγεγραμμένοι χρήστες μπορούν να συνδεθούν στην εφαρμογή χρησιμοποιώντας το email και τον κωδικό τους.    
Προφίλ χρήστη: Οι συνδεδεμένοι χρήστες μπορούν να δουν το προφίλ τους, το ιστορικό κρατήσεων και να αποσυνδεθούν.    
Αναζήτηση εστιατορίων: Οι χρήστες μπορούν να αναζητήσουν εστιατόρια με βάση το όνομα και την τοποθεσία.    
Κράτηση εστιατορίου: Οι χρήστες μπορούν να κάνουν κράτηση σε ένα εστιατόριο επιλέγοντας ημερομηνία, ώρα και αριθμό ατόμων.    
Αποσύνδεση: Οι χρήστες μπορούν να αποσυνδεθούν από την εφαρμογή.    
Τεχνολογίες που χρησιμοποιούνται:

Backend: Node.js, Express, MySQL, JWT, bcryptjs, cors, body-parser, cookie-parser, dotenv
Frontend: React, axios, react-router-dom, jwt-decode
CSS: Χειροποίητο CSS (App.css) 

Κώδικας όλης της εργασίας:

BACKEND:

.env αρχείο:
JWT_SECRET=supersecretkey 

backend.js:
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const mysql = require('mysql2/promise');
const cookieParser = require('cookie-parser');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 5000;
const SECRET_KEY = process.env.JWT_SECRET;

if (!SECRET_KEY) {
    console.error('❌ Missing JWT_SECRET in environment variables');
    process.exit(1);
}

// Middleware
app.use(cors({
    origin: 'http://localhost:3000',
    credentials: true
}));
app.use(bodyParser.json());
app.use(cookieParser());

// Database connection
const db = mysql.createPool({
    host: 'localhost',
    user: 'root',
    password: '',
    database: 'restaurantprogram',
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0
});

// Check Database Connection
db.getConnection()
    .then(() => console.log('✅ Connected to the database'))
    .catch(err => {
        console.error('❌ Could not connect to the database', err);
        process.exit(1);
    });

// Εγγραφή χρήστη
app.post('/register', async (req, res) => {
    try {
        const { name, email, password } = req.body;
        if (!name || !email || !password) {
            return res.status(400).json({ message: 'Όλα τα πεδία είναι υποχρεωτικά' });
        }

        const [existingUsers] = await db.execute('SELECT * FROM users WHERE email = ?', [email]);
        if (existingUsers.length > 0) {
            return res.status(400).json({ message: 'Το email χρησιμοποιείται ήδη' });
        }

        const hashedPassword = await bcrypt.hash(password, 10);
        const [result] = await db.execute('INSERT INTO users (name, email, password) VALUES (?, ?, ?)', [name, email, hashedPassword]);

        const userId = result.insertId;
        const token = jwt.sign({ userId, name, email }, SECRET_KEY, { expiresIn: '1h' });

        // Για τοπικό περιβάλλον, secure: false, για παραγωγή secure: true.
        res.cookie('token', token, { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'None' });
        return res.status(201).json({ message: 'Επιτυχής εγγραφή!', token });
    } catch (error) {
        console.error(error);
        return res.status(500).json({ message: 'Σφάλμα κατά την εγγραφή', details: error.message });
    }
});

// Σύνδεση χρήστη (Login)
app.post('/login', async (req, res) => {
    try {
        const { email, password } = req.body;
        if (!email || !password) {
            return res.status(400).json({ message: 'Όλα τα πεδία είναι υποχρεωτικά' });
        }

        const [users] = await db.execute('SELECT * FROM users WHERE email = ?', [email]);
        if (users.length === 0) {
            return res.status(400).json({ message: 'Λάθος email ή κωδικός' });
        }

        const user = users[0];
        const passwordMatch = await bcrypt.compare(password, user.password);
        if (!passwordMatch) {
            return res.status(400).json({ message: 'Λάθος email ή κωδικός' });
        }

        const token = jwt.sign({ userId: user.id, name: user.name, email }, SECRET_KEY, { expiresIn: '1h' });

        res.cookie('token', token, { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'None' });
        return res.status(200).json({ message: 'Επιτυχής σύνδεση!', token });
    } catch (error) {
        console.error(error);
        return res.status(500).json({ message: 'Σφάλμα κατά τη σύνδεση', details: error.message });
    }
});

// Endpoint για αναζήτηση εστιατορίων
app.get('/restaurants', async (req, res) => {
    try {
        const { name = '', location = '' } = req.query;
        
        let query = 'SELECT * FROM restaurants WHERE 1';
        const queryParams = [];

        if (name) {
            query += ' AND name LIKE ?';
            queryParams.push(`%${name}%`);
        }
        if (location) {
            query += ' AND location LIKE ?';
            queryParams.push(`%${location}%`);
        }

        const [restaurants] = await db.execute(query, queryParams);
        return res.status(200).json(restaurants);
    } catch (error) {
        console.error('Σφάλμα κατά την αναζήτηση εστιατορίων:', error);
        return res.status(500).json({ error: 'Σφάλμα κατά την αναζήτηση εστιατορίων.' });
    }
});

// Backend endpoint για κρατήσεις (GET)
app.get('/reservations', async (req, res) => {
    try {
      const token = req.headers.authorization?.split(' ')[1];
      if (!token) {
        return res.status(401).json({ error: 'Μη εξουσιοδοτημένος' });
      }
  
      const decoded = jwt.verify(token, SECRET_KEY);
      const userId = decoded.userId;
  
      // Ερώτημα για τις κρατήσεις του χρήστη
      const [bookings] = await db.execute(
        'SELECT * FROM reservations WHERE user_id = ?',
        [userId]
      );
  
      res.status(200).json(bookings);
    } catch (error) {
      console.error('Σφάλμα κατά την ανάκτηση των κρατήσεων:', error);
      return res.status(500).json({ error: 'Σφάλμα κατά την ανάκτηση των κρατήσεων.' });
    }
  });
  
  app.post('/reservations', async (req, res) => {
    try {
      const token = req.headers.authorization?.split(' ')[1];
      if (!token) {
        return res.status(401).json({ error: 'Μη εξουσιοδοτημένος' });
      }
  
      const decoded = jwt.verify(token, SECRET_KEY);
      const { restaurant_id, date, time, people_count } = req.body;
  
      if (!restaurant_id || !date || !time || !people_count) {
        return res.status(400).json({ error: 'Όλα τα πεδία είναι υποχρεωτικά' });
      }
  
      console.log('Decoded Token:', decoded); // Ελέγχουμε αν το decoded token είναι σωστό
      await db.execute(
        'INSERT INTO reservations (user_id, restaurant_id, date, time, people_count) VALUES (?, ?, ?, ?, ?)',
        [decoded.userId, restaurant_id, date, time, people_count]
      );
  
      return res.status(201).json({ message: 'Η κράτηση πραγματοποιήθηκε με επιτυχία!' });
    } catch (error) {
      console.error('Σφάλμα κατά την κράτηση:', error); // Ενίσχυση της καταγραφής
      return res.status(500).json({ error: 'Σφάλμα κατά την κράτηση.' });
    }
  });
  



// Server startup
app.listen(PORT, () => {
    console.log(`🚀 Server running on port ${PORT}`);
});

FRONTEND

App.css:
/* App.css */
body {
  font-family: 'Poppins', sans-serif;
  background-color: white;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  margin: 0;
}

h1, h2, h3, p, tr, td{
  color: #159af3; /* Γαλάζιο */
  text-align: center;
}

button {
  background-color: #169df8; /* Γαλάζιο */
  color: white;
  border: none;
  padding: 10px 20px;
  margin: 10px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

button:hover {
  background-color: #30a3ef;
}

input {
  padding: 10px;
  margin: 10px;
  border-radius: 5px;
  border: 1px solid #ddd;
  width: 100%;
  max-width: 300px;
  font-size: 16px;
}

form {
  display: flex;
  flex-direction: column;
  align-items: center;
}

img {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 20px auto;
}

.container {
  text-align: center;
  max-width: 800px;
  margin: 20px;
}

App.js:
import React from 'react';
import { BrowserRouter as Router, Route, Routes, useNavigate } from 'react-router-dom';
import './App.css'; // Εισαγωγή του CSS
// Εισαγωγή των Components
import Register from './Register';
import Login from './Login';
import UserProfile from './UserProfile';
import BookingForm from './BookingForm';
import RestaurantList from './RestaurantList';

const Home = () => {
  const navigate = useNavigate();

  return (
    <div>
      <h1>Αρχική Σελίδα</h1><br />
      <h1>Καλώς ήρθατε στα Ελληνικά εστιατόρια!!</h1><br />
      <img 
        src={require('./elliniki-simaia-1-eur.webp')}  
        alt="Εικόνα 1" 
        style={{ width: '400px', height: '200px', objectFit: 'cover' }} 
      />
      <img 
        src={require('./1376387.png')} 
        alt="Εικόνα 2" 
        style={{ width: '200px', height: '200px', objectFit: 'cover' }} 
      /><br></br><br></br>
      <button onClick={() => navigate('/register')}>Εγγραφή</button>
      <button onClick={() => navigate('/login')}>Σύνδεση</button>
    </div>
  );
};

const App = () => {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/register" element={<Register />} />
        <Route path="/login" element={<Login />} />
        <Route path="/user-profile" element={<UserProfile />} />
        <Route path="/booking/:restaurantId" element={<BookingForm />} />
        <Route path="/restaurants" element={<RestaurantList />} />
        <Route path="*" element={<h2>404: Σελίδα δεν βρέθηκε</h2>} />
      </Routes>
    </Router>
  );
};

export default App;

Login.js:
import React, { useState } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';
import './App.css'; // Εισαγωγή του CSS
const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleLogin = async (e) => {
    e.preventDefault();
    setError('');

    try {
      const response = await axios.post(
        'http://localhost:5000/login',
        { email, password },
        { headers: { 'Content-Type': 'application/json' }, withCredentials: true }
      );

      console.log('Απάντηση διακομιστή:', response.data);

      if (response.data.token) {
        localStorage.setItem('token', response.data.token);
        navigate('/user-profile');
      } else {
        throw new Error('Λάθος απάντηση από τον διακομιστή');
      }
    } catch (err) {
      console.error('Σφάλμα κατά τη σύνδεση:', err.response || err);
      setError(err.response?.data?.message || 'Πρόβλημα σύνδεσης στον διακομιστή');
    }
  };

  return (
    <div>
      <h1>Σύνδεση</h1>
      <form onSubmit={handleLogin}>
        <input
          type="email"
          placeholder="Ηλεκτρονικό Ταχυδρομείο"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
        <input
          type="password"
          placeholder="Κωδικός Πρόσβασης"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
        <button type="submit">Σύνδεση</button>
      </form>
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <button onClick={() => navigate('/register')}>
        Δεν έχεις λογαριασμό; Εγγραφή
      </button>
    </div>
  );
};

export default Login;

UserProfile.js:
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';
import { jwtDecode } from 'jwt-decode';
import './App.css'; // Εισαγωγή του CSS
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const [bookings, setBookings] = useState([]);
  const [message, setMessage] = useState('');
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (!token) {
      navigate('/login');
      return;
    }

    try {
      const decoded = jwtDecode(token);
      console.log("Decoded Token:", decoded);
      setUser({ user_id: decoded.userId, name: decoded.name });

      const fetchBookings = async () => {
        try {
          const response = await axios.get(`http://localhost:5000/reservations`, {
            headers: { Authorization: `Bearer ${token}` },
          });
          console.log("Bookings:", response.data);
          setBookings(response.data);
        } catch (error) {
          console.error("Σφάλμα κατά την ανάκτηση των κρατήσεων:", error);
          setMessage(error.response?.data?.error || 'Σφάλμα κατά την ανάκτηση των κρατήσεων.');
        }
      };

      fetchBookings();
    } catch (err) {
      console.error("Σφάλμα αποκωδικοποίησης token:", err);
      navigate('/login');
    }
  }, [navigate]);

  const handleLogout = () => {
    localStorage.removeItem('token');
    navigate('/login');
  };

  if (!user) return <p>{message || 'Δεν βρέθηκε το προφίλ του χρήστη.'}</p>;

  return (
    <div>
      <h2>Προφίλ Χρήστη</h2>
      <p><strong>Όνομα:</strong> {user.name || 'Δεν υπάρχει όνομα'}</p>

      {message && <p style={{ color: 'red' }}>{message}</p>}

      <button onClick={handleLogout}>Αποσύνδεση</button>
      <button onClick={() => navigate('/restaurants')}>Δες Εστιατόρια</button>

      <h3>Ιστορικό Κρατήσεων</h3>
      {bookings.length === 0 ? (
        <p>Δεν έχετε καμία κράτηση.</p>
      ) : (
        <table style={{ width: '100%', borderCollapse: 'collapse' }}>
  <thead>
    <tr style={{ backgroundColor: '#f2f2f2' }}>
      <th style={{ border: '1px solid #ddd', padding: '8px', textAlign: 'left' }}>Ημερομηνία</th>
      <th style={{ border: '1px solid #ddd', padding: '8px', textAlign: 'left' }}>Ώρα</th>
      <th style={{ border: '1px solid #ddd', padding: '8px', textAlign: 'left' }}>Αριθμός Ατόμων</th>
    </tr>
  </thead>
  <tbody>
    {bookings.map((booking) => (
      <tr key={booking.reservation_id}>
        <td style={{ border: '1px solid #ddd', padding: '8px' }}>{booking.date}</td>
        <td style={{ border: '1px solid #ddd', padding: '8px' }}>{booking.time}</td>
        <td style={{ border: '1px solid #ddd', padding: '8px' }}>{booking.people_count}</td>
      </tr>
    ))}
  </tbody>
</table>

      )}
    </div>
  );
};

export default UserProfile;

RestaurantList.js:
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';
import './App.css'; // Εισαγωγή του CSS
const RestaurantList = () => {
  const [restaurants, setRestaurants] = useState([]);
  const [searchName, setSearchName] = useState('');
  const [searchLocation, setSearchLocation] = useState('');
  
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (!token) {
      navigate('/login');
    }
  }, [navigate]);

  const fetchRestaurants = async () => {
    try {
      const response = await axios.get('http://localhost:5000/restaurants', {
        params: {
          name: searchName,
          location: searchLocation
        },
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      });
      setRestaurants(response.data);
    } catch (error) {
      console.error('Σφάλμα κατά την ανάκτηση των εστιατορίων:', error);
    }
  };

  const handleLogout = () => {
    localStorage.removeItem('token');
    navigate('/login');
  };

  return (
    <div>
      <h2>Κατάλογος εστιατορίων</h2>
      <div>
        <input
          type="text"
          placeholder="Όνομα Εστιατορίου"
          value={searchName}
          onChange={(e) => setSearchName(e.target.value)}
        />
        <input
          type="text"
          placeholder="Τοποθεσία"
          value={searchLocation}
          onChange={(e) => setSearchLocation(e.target.value)}
        />
        <button onClick={fetchRestaurants}>Αναζήτηση</button>
      </div>
      <ul>
        {restaurants.length === 0 ? (
          <li>Δεν βρέθηκαν εστιατόρια</li>
        ) : (
          restaurants.map((restaurant) => (
            <li key={restaurant.restaurant_id}>
              <h3>{restaurant.name}</h3>
              <p>{restaurant.location}</p>
              <p>{restaurant.description}</p>
              <button onClick={() => navigate(`/booking/${restaurant.restaurant_id}`)}>Κάνε Κράτηση</button>
            </li>
          ))
        )}
      </ul>
      <button onClick={() => navigate('/user-profile')}>Προφίλ Χρήστη</button>
      <button onClick={handleLogout}>Αποσύνδεση</button>
    </div>
  );
};

export default RestaurantList;

BookingForm.js:
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useNavigate, useParams } from 'react-router-dom';
import { jwtDecode } from 'jwt-decode'; // Χρησιμοποιούμε την σωστή σύνταξη για την εισαγωγή
import './App.css'; // Εισαγωγή του CSS
const BookingForm = () => {
  const { restaurantId } = useParams();
  const [date, setDate] = useState('');
  const [time, setTime] = useState('');
  const [numPeople, setNumPeople] = useState(1);
  const [message, setMessage] = useState('');
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (!token) {
      navigate('/login');
    }
  }, [navigate]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const token = localStorage.getItem('token');
    if (!token) {
      setMessage('Πρέπει να είστε συνδεδεμένοι για να κάνετε κράτηση');
      return;
    }
  
    try {
      const decodedToken = jwtDecode(token);
      const userId = decodedToken.userId;
  
      if (!userId) {
        setMessage('Σφάλμα: Δεν βρέθηκε userId στο token');
        return;
      }
  
      const response = await axios.post(
        'http://localhost:5000/reservations',
        {
          restaurant_id: restaurantId,
          date,
          time,
          people_count: numPeople,
          user_id: userId
        },
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
  
      setMessage(response.data?.message || 'Η κράτησή σας καταχωρήθηκε επιτυχώς!');
    } catch (error) {
      console.error('Σφάλμα κατά την κράτηση:', error);
  
      // Χειρισμός του σφάλματος
      if (error.response) {
        setMessage(error.response?.data?.error || 'Σφάλμα κατά την κράτηση.');
      } else {
        setMessage('Σφάλμα κατά την επικοινωνία με τον server.');
      }
    }
  };
  

  const handleLogout = () => {
    localStorage.removeItem('token');
    navigate('/login');
  };

  return (
    <div>
      <h2>Φόρμα Κράτησης</h2>
      <form onSubmit={handleSubmit}>
        <input
          type="date"
          value={date}
          onChange={(e) => setDate(e.target.value)}
          required
        />
        <input
          type="time"
          value={time}
          onChange={(e) => setTime(e.target.value)}
          required
        />
        <input
          type="number"
          value={numPeople}
          onChange={(e) => setNumPeople(Number(e.target.value))}
          min="1"
          required
        />
        <button type="submit">Κράτηση</button>
      </form>
      {message && <p>{message}</p>}

      <button onClick={() => navigate('/restaurants')}>Πίσω στα Εστιατόρια</button>
      <button onClick={() => navigate('/user-profile')}>Προφίλ Χρήστη</button>
      <button onClick={handleLogout}>Αποσύνδεση</button>
    </div>
  );
};

export default BookingForm;

Register.js:
import React, { useState } from 'react';
import axios from 'axios';
import { useNavigate } from 'react-router-dom';
import './App.css'; // Εισαγωγή του CSS
const Register = () => {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [message, setMessage] = useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post('http://localhost:5000/register', { name, email, password }, { withCredentials: true });
      setMessage(response.data.message || 'Επιτυχής εγγραφή!');
      setTimeout(() => navigate('/login'), 2000);
    } catch (error) {
      setMessage(error.response?.data?.message || 'Αποτυχία εγγραφής, παρακαλώ δοκιμάστε πάλι.');
    }
  };
  

  return (
    <div>
      <h2>Εγγραφή</h2>
      <form onSubmit={handleSubmit}>
        <input type="text" placeholder="Όνομα" value={name} onChange={(e) => setName(e.target.value)} required />
        <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        <input type="password" placeholder="Κωδικός" value={password} onChange={(e) => setPassword(e.target.value)} required />
        <button type="submit">Εγγραφή</button>
      </form>
      {message && <p>{message}</p>}
      <button onClick={() => navigate('/login')}>Ήδη έχεις λογαριασμό; Σύνδεση</button>
    </div>
  );
};

export default Register;

