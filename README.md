# Firebase With React

## Firebase Setup

1. Create a firebase project
1. Create web app inside your firebase project
1. Install firebase using 'npm i firebase'
1. Create a config file in your project, make sure to put variables inside an env file

   ```javascript
   import { initializeApp } from 'firebase/app'
   import { getFirestore } from 'firebase/firestore'

   const firebaseConfig = {
     apiKey: process.env.REACT_APP_FIREBASE_API_KEY,
     authDomain: process.env.REACT_APP_FIREBASE_AUTH_DOMAIN,
     projectId: process.env.REACT_APP_FIREBASE_PROJECT_ID,
     storageBucket: process.env.REACT_APP_FIREBASE_STORAGE_BUCKET,
     messagingSenderId: process.env.REACT_APP_FIREBASE_MESSAGING_SENDER_ID,
     appId: process.env.REACT_APP_FIREBASE_APP_ID,
   }

   initializeApp(firebaseConfig)
   export const db = getFirestore()
   ```

<br>

## Enable Authentication

To enable authentication to go authentication tab and enable providers for the methods you want to use like email/password

<br>

## Firestore Database Rules

```javascript
service cloud.firestore {
  match /databases/{database}/documents {


    function isSignedIn() {
      return request.auth != null;
    }

    function emailVerified() {
      return request.auth.token.email_verified;
    }

    function userExists() {
      return exists(/databases/$(database)/documents/users/$(request.auth.uid));
    }

    // [READ] Data that exists on the Firestore document
    function existingData() {
      return resource.data;
    }

    // [WRITE] Data that is sent to a Firestore document
    function incomingData() {
      return request.resource.data;
    }

   // Does the logged-in user match the requested userId
    function isOwner(userId) {
      return request.auth.uid == userId;
    }

    // Fetch a user from Firestore
    function getUserData() {
      return get(/databases/$(database)/documents/accounts/$(request.auth.uid)).data
    }

    // Fetch a user-specific field from Firestore
    function userEmail(userId) {
      return get(/databases/$(database)/documents/users/$(userId)).data.email;
    }


    // example application for functions
    match /orders/{orderId} {
      allow create: if isSignedIn() && emailVerified() && isUser(incomingData().userId);
      allow read, list, update, delete: if isSignedIn() && isUser(existingData().userId);
    }

  }
}
```

<br>

## Storage Rules

```javascript
service firebase.storage {
  match /b/{bucket}/o {

  	// Rules
    match /{allPaths=**} {
      allow read;
      allow write: if
      isSignedIn() &&
      incomingData().size < 2 * 1024 * 102 &&
      incomingData().contentType.matches('image/.*')
    }

    // Functions
    function incomingData() {
      return request.resource.data;
    }
  }
}
```

<br>

## Register User

```javascript
import {
  getAuth,
  createUserWithEmailAndPassword,
  updateProfile,
} from 'firebase/auth'
import { setDoc, doc, serverTimestamp } from 'firebase/firestore'

// on submitting user data
const auth = getAuth()
const userCredential = await createUserWithEmailAndPassword(
  auth,
  email,
  password
)
const user = userCredential.user
updateProfile(auth.currentUser, {
  displayName: name,
})
// saving user to firestore database
const formDataCopy = { ...formData }
delete formDataCopy.password
formDataCopy.timestamp = serverTimestamp()
await setDoc(doc(db, 'users', user.uid), formDataCopy)
```

<br>

## Login User

```javascript
import { getAuth, signInWithEmailAndPassword } from 'firebase/auth'

// on submitting user data
const auth = getAuth()
const userCredential = await signInWithEmailAndPassword(auth, email, password)
```

<br>

## Logout

```javascript
import { getAuth } from 'firebase/auth'
const auth = getAuth()
auth.signOut()
```

<br>

## Forgot Password

```javascript
import { getAuth, sendPasswordResetEmail } from 'firebase/auth'
// on submitting user data
const auth = getAuth()
await sendPasswordResetEmail(auth, email)
```

<br>

## Google OAuth

```javascript
import { getAuth, signInWithPopup, GoogleAuthProvider } from 'firebase/auth'
import { doc, setDoc, getDoc, serverTimestamp } from 'firebase/firestore'
import { db } from '../firebase.config'

//on google click
const auth = getAuth()
const provider = new GoogleAuthProvider()
const result = await signInWithPopup(auth, provider)
const user = result.user

// Check for user
const docRef = doc(db, 'users', user.uid)
const docSnap = await getDoc(docRef)

// If user, doesn't exist, create user
if (!docSnap.exists()) {
  await setDoc(doc(db, 'users', user.uid), {
    name: user.displayName,
    email: user.email,
    timestamp: serverTimestamp(),
  })
}
```

<br>

## Login with Facebook

Go to [Facebook for Developers](https://developers.facebook.com/). Head to my apps and selected consumer and finish. Once the app is created to go settings > basic settings and copy the **app id** and **app secret** to firebase provider. Copy the OAuth redirect URI. Go to dashboard setup facebook login. Under valid OAuth redirect URI paste.

```js
import { signInWithPopup, FacebookAuthProvider } from 'firebase/auth'

const provider = new FacebookAuthProvider()
```

<br>

## Private Routes

### useAuthStatus.js

```javascript
import { useEffect, useState, useRef } from 'react'
import { getAuth, onAuthStateChanged } from 'firebase/auth'

export const useAuthStatus = () => {
  const [loggedIn, setLoggedIn] = useState(false)
  const [checkingStatus, setCheckingStatus] = useState(true)
  const isMounted = useRef(true)

  useEffect(() => {
    if (isMounted) {
      const auth = getAuth()
      onAuthStateChanged(auth, user => {
        if (user) {
          setLoggedIn(true)
        }
        setCheckingStatus(false)
      })
    }

    return () => {
      isMounted.current = false
    }
  }, [isMounted])

  return { loggedIn, checkingStatus }
}
```

### PrivateRoute.js

```javascript
import { Navigate, Outlet } from 'react-router-dom'
import { useAuthStatus } from '../hooks/useAuthStatus'

const PrivateRoute = () => {
  const { loggedIn, checkingStatus } = useAuthStatus()

  if (checkingStatus) {
    return <h3>Loading...</h3>
  }

  return loggedIn ? <Outlet /> : <Navigate to='/sign-in' />
}

export default PrivateRoute
```

<br>

## Fetch Collections from Database

```javascript
import {
  collection,
  getDocs,
  query,
  where,
  orderBy,
  limit,
  startAfter,
} from 'firebase/firestore'
import { db } from '../firebase.config'

// when components load
// Get reference
const productsRef = collection(db, 'products')

// Create a query
const q = query(
  productsRef,
  where('type', '==', categoryName),
  orderBy('timestamp', 'desc'),
  limit(10)
)

// Execute query
const querySnap = await getDocs(q)
const products = []
querySnap.forEach(doc => {
  return products.push({
    id: doc.id,
    data: doc.data(),
  })
})
```

<br>

## Fetch Document from Collection

```javascript
import { getDoc, doc } from 'firebase/firestore'
import { getAuth } from 'firebase/auth'
import { db } from '../firebase.config'

//on component load
const docRef = doc(db, 'products', productId) //productId from params
const docSnap = await getDoc(docRef)
// docSnap to your state after checking it exists
```

<br>

## Uploading Images

```javascript
import {
  getStorage,
  ref,
  uploadBytesResumable,
  getDownloadURL,
} from 'firebase/storage'
import { db } from '../firebase.config'

// Store image in firebase
const storeImage = async image => {
  return new Promise((resolve, reject) => {
    const storage = getStorage()
    const fileName = `${auth.currentUser.uid}-${image.name}-${uuidv4()}`

    const storageRef = ref(storage, 'images/' + fileName)

    const uploadTask = uploadBytesResumable(storageRef, image)

    uploadTask.on(
      'state_changed',
      snapshot => {
        const progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100
        console.log('Upload is ' + progress + '% done')
        switch (snapshot.state) {
          case 'paused':
            console.log('Upload is paused')
            break
          case 'running':
            console.log('Upload is running')
            break
          default:
            break
        }
      },
      error => {
        reject(error)
      },
      () => {
        // Handle successful uploads on complete
        // For instance, get the download URL: https://firebasestorage.googleapis.com/...
        getDownloadURL(uploadTask.snapshot.ref).then(downloadURL => {
          resolve(downloadURL)
        })
      }
    )
  })
}

const imgUrls = await Promise.all(
  [...images].map(image => storeImage(image)) //images is the state
).catch(() => {
  setLoading(false)
  console.error('Images not uploaded')
  return
})

const handleChange = e => {
  if (e.target.files) {
    setFormData(prevState => ({
      ...prevState,
      images: e.target.files,
    }))
  }
}

return (
  <input
    type='file'
    onChange={handleChange}
    max='6'
    accept='.jpg,.png,.jpeg'
    multiple
    required
  />
)
```

<br>

## Create Document in a Collection

```javascript
import { addDoc, collection, serverTimestamp } from 'firebase/firestore'

//on submit data
const docRef = await addDoc(collection(db, 'products'), formData)
//docRef gives id too if you want to navigate to that
```

<br>

## Delete Document in a Collection

```javascript
import { addDoc, doc } from 'firebase/firestore'

//on delete
await deleteDoc(doc(db, 'products', productId))
```

<br>

## Update Document in a Collection

```javascript
import { updateDoc, doc } from 'firebase/firestore'

//on update
await updateDoc(doc(db, 'products'), {
  onSale: false,
})
```

<br>
