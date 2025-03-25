# MERN application that integrates TensorFlow
Creating a MERN application that integrates TensorFlow for document scanning and allows users to authenticate, add, edit, delete, and store scanned documents in MongoDB involves combining several technologies. Below is a detailed plan and implementation strategy:

# Application Features
- User Authentication:
   - Register and login functionality using JWT (JSON Web Tokens).
   - Password encryption using bcrypt.

- Document Management
  - Scan documents using TensorFlow-based Optical Character Recognition (OCR).
  - Add, edit, delete, and fetch documents.

- Storage
  - Store documents and their metadata in MongoDB.
  - Store scanned document images in a directory or cloud storage (e.g., AWS S3).

 - Frontend:
  - Use React for a clean, responsive user interface.

# Technology Stack

 - Frontend: React.js
 - Backend: Node.js with Express.js
 - Database: MongoDB
 - AI/ML: TensorFlow.js or TensorFlow Python server
 - Authentication: JWT with bcrypt

# Implementation Steps
Setup Backend

- Install Dependencies

```bash
npm install express mongoose bcrypt jsonwebtoken body-parser multer
npm install @tensorflow/tfjs-node

```
Server (server.js)
```javascript
const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const multer = require('multer');
const tf = require('@tensorflow/tfjs-node');
const { OCRModel } = require('./ocr');

const app = express();
app.use(express.json());

// MongoDB connection
mongoose.connect('mongodb://localhost:27017/mern-docs', {
    useNewUrlParser: true,
    useUnifiedTopology: true
});

// User schema and model
const UserSchema = new mongoose.Schema({
    username: String,
    password: String,
});
const User = mongoose.model('User', UserSchema);

// Document schema and model
const DocumentSchema = new mongoose.Schema({
    userId: mongoose.Types.ObjectId,
    title: String,
    content: String,
    filePath: String,
});
const Document = mongoose.model('Document', DocumentSchema);

// Middleware: Authenticate User
const authenticate = (req, res, next) => {
    const token = req.headers['authorization'];
    if (!token) return res.status(401).send('Unauthorized');

    try {
        const user = jwt.verify(token, 'SECRET_KEY');
        req.user = user;
        next();
    } catch (err) {
        res.status(401).send('Invalid Token');
    }
};

// Routes
app.post('/register', async (req, res) => {
    const hashedPassword = await bcrypt.hash(req.body.password, 10);
    const user = new User({ username: req.body.username, password: hashedPassword });
    await user.save();
    res.send('User Registered');
});

app.post('/login', async (req, res) => {
    const user = await User.findOne({ username: req.body.username });
    if (!user || !(await bcrypt.compare(req.body.password, user.password))) {
        return res.status(401).send('Invalid Credentials');
    }
    const token = jwt.sign({ id: user._id }, 'SECRET_KEY');
    res.json({ token });
});

// File upload setup
const storage = multer.diskStorage({
    destination: './uploads/',
    filename: (req, file, cb) => cb(null, Date.now() + '-' + file.originalname),
});
const upload = multer({ storage });

// OCR with TensorFlow
app.post('/scan', upload.single('file'), authenticate, async (req, res) => {
    const model = await OCRModel.load(); // Load TensorFlow OCR model
    const content = await model.scan(req.file.path);
    const document = new Document({
        userId: req.user.id,
        title: req.body.title,
        content,
        filePath: req.file.path,
    });
    await document.save();
    res.json({ document });
});

app.get('/documents', authenticate, async (req, res) => {
    const documents = await Document.find({ userId: req.user.id });
    res.json(documents);
});

app.put('/documents/:id', authenticate, async (req, res) => {
    const document = await Document.findByIdAndUpdate(req.params.id, req.body, { new: true });
    res.json(document);
});

app.delete('/documents/:id', authenticate, async (req, res) => {
    await Document.findByIdAndDelete(req.params.id);
    res.send('Document Deleted');
});

app.listen(3000, () => console.log('Server running on port 3000'));

```

# TensorFlow OCR Implementation
OCR Model Loader (ocr.js)
```javascript
const tf = require('@tensorflow/tfjs-node');
const fs = require('fs');

class OCRModel {
    static async load() {
        if (!this.model) {
            this.model = await tf.loadLayersModel('file://path-to-ocr-model/model.json');
        }
        return this;
    }

    async scan(filePath) {
        const image = fs.readFileSync(filePath);
        const tensor = tf.node.decodeImage(image, 3).expandDims(0); // Adjust as per model input
        const prediction = this.model.predict(tensor);
        const text = prediction.dataSync().join('');
        return text;
    }
}

module.exports = { OCRModel };

```



# Frontend in React
Login and Register (React)

- Create login and register components using React Hook Form.
- Use Axios for API calls.

Document Management (React)

 - Create a component to upload, edit, delete, and display documents.
 - Use React hooks to manage state.

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const DocumentManager = () => {
    const [documents, setDocuments] = useState([]);
    const [file, setFile] = useState(null);

    const fetchDocuments = async () => {
        const response = await axios.get('/documents', {
            headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
        });
        setDocuments(response.data);
    };

    const uploadFile = async () => {
        const formData = new FormData();
        formData.append('file', file);
        const response = await axios.post('/scan', formData, {
            headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
        });
        console.log(response.data);
        fetchDocuments();
    };

    useEffect(() => {
        fetchDocuments();
    }, []);

    return (
        <div>
            <h1>Document Manager</h1>
            <input type="file" onChange={(e) => setFile(e.target.files[0])} />
            <button onClick={uploadFile}>Upload</button>
            <ul>
                {documents.map((doc) => (
                    <li key={doc._id}>{doc.title}</li>
                ))}
            </ul>
        </div>
    );
};

export default DocumentManager;

```


# React Components

## Login Component

This component handles user login and sets the JWT token in local storage.

```javascript

import React, { useState } from 'react';
import axios from 'axios';

const Login = ({ setIsAuthenticated }) => {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const [error, setError] = useState('');

    const handleLogin = async () => {
        try {
            const response = await axios.post('http://localhost:3000/login', {
                username,
                password
            });
            localStorage.setItem('token', response.data.token);
            setIsAuthenticated(true);
        } catch (err) {
            setError('Invalid username or password');
        }
    };

    return (
        <div>
            <h1>Login</h1>
            <input
                type="text"
                placeholder="Username"
                value={username}
                onChange={(e) => setUsername(e.target.value)}
            />
            <input
                type="password"
                placeholder="Password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
            />
            <button onClick={handleLogin}>Login</button>
            {error && <p style={{ color: 'red' }}>{error}</p>}
        </div>
    );
};

export default Login;
```

 ## Document Manager
 
This component lets users upload scanned files, view document metadata, and delete documents.

```javascript

import React, { useState, useEffect } from 'react';
import axios from 'axios';

const DocumentManager = () => {
    const [documents, setDocuments] = useState([]);
    const [file, setFile] = useState(null);
    const [title, setTitle] = useState('');
    const [error, setError] = useState('');

    const fetchDocuments = async () => {
        const token = localStorage.getItem('token');
        const response = await axios.get('http://localhost:3000/documents', {
            headers: { Authorization: `Bearer ${token}` },
        });
        setDocuments(response.data);
    };

    const uploadDocument = async () => {
        if (!file || !title) {
            setError('Please provide both a file and a title');
            return;
        }

        const formData = new FormData();
        formData.append('file', file);
        formData.append('title', title);

        try {
            const token = localStorage.getItem('token');
            await axios.post('http://localhost:3000/scan', formData, {
                headers: { Authorization: `Bearer ${token}` },
            });
            fetchDocuments();
        } catch (err) {
            setError('Failed to upload document');
        }
    };

    const deleteDocument = async (id) => {
        const token = localStorage.getItem('token');
        await axios.delete(`http://localhost:3000/documents/${id}`, {
            headers: { Authorization: `Bearer ${token}` },
        });
        fetchDocuments();
    };

    useEffect(() => {
        fetchDocuments();
    }, []);

    return (
        <div>
            <h1>Document Manager</h1>
            <input
                type="text"
                placeholder="Document Title"
                value={title}
                onChange={(e) => setTitle(e.target.value)}
            />
            <input
                type="file"
                onChange={(e) => setFile(e.target.files[0])}
            />
            <button onClick={uploadDocument}>Upload</button>
            {error && <p style={{ color: 'red' }}>{error}</p>}
            <ul>
                {documents.map((doc) => (
                    <li key={doc._id}>
                        {doc.title}{' '}
                        <button onClick={() => deleteDocument(doc._id)}>Delete</button>
                    </li>
                ))}
            </ul>
        </div>
    );
};

export default DocumentManager;
```

## Deployment Instructions

# Backend Deployment

 - Use Docker
Create a Dockerfile for the Node.js backend:

```dockerfile
# Use Node.js base image
FROM node:16

# Set the working directory
WORKDIR /app

# Copy package.json and install dependencies
COPY package.json package-lock.json ./
RUN npm install

# Copy the rest of the application
COPY . .

# Expose port and run the server
EXPOSE 3000
CMD ["node", "server.js"]

```

Build and run the container:

```bash
docker build -t mern-backend .
docker run -p 3000:3000 mern-backend
```

  - Deploy to AWS Elastic Beanstalk
Install the Elastic Beanstalk CLI

```bash
pip install awsebcli
```

  - Initialize Elastic Beanstalk

```bash
eb init
```

  - Deploy the application
    
```bash
eb create my-backend-env
```

# Frontend Deployment

  - Build the React Application
Run the following command in the React project directory:

```bash
npm run build
```

This creates a production-ready build directory.

 - Deploy to AWS S3

   - Create an S3 Bucket:
    Enable static website hosting.
   - Upload the Build Folder:
   Use the AWS Management Console or the AWS CLI:

```bash
aws s3 sync build/ s3://your-bucket-name
```

   - Set Permissions:
     - Allow public access to the bucket.
     - Update the S3 bucket's policy to allow static site hosting.
    
  #   MongoDB Deployment
Use MongoDB Atlas for a cloud database:

  - Create a cluster on MongoDB Atlas.
  - Update the connection string in server.js
    
```bash
    mongoose.connect('your-mongodb-atlas-connection-string', {
    useNewUrlParser: true,
    useUnifiedTopology: true
});
```
#  TensorFlow Model Deployment

  - Host Model
Host the TensorFlow model on an S3 bucket or a file server.

  - Update Backend
Update the backend to load the model from the hosted location:

```javascript
const model = await tf.loadLayersModel('https://your-s3-bucket/model.json');
```
This setup covers both local development and cloud deployment of the MERN application integrated with TensorFlow for document scanning.

# Enhancements


 - Cloud Storage:
   - Use AWS S3 or similar for storing scanned documents.
 - Deployment:
   - Use Docker for containerization.
   - Deploy using AWS Elastic Beanstalk or Heroku.
 - Advanced OCR:
   - Integrate Google Vision API or AWS Textract for more robust scanning.






 # License
This project is licensed under the MIT License.
