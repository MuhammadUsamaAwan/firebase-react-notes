# Firebase With React

## Firebase Setup

```js
import { initializeApp } from 'firebase/app'
import { getFirestore } from 'firebase/firestore'
import { getStorage } from 'firebase/storage'
import { getAuth } from 'firebase/auth'

const firebaseConfig = {
  apiKey: process.env.REACT_APP_FIREBASE_API_KEY,
  authDomain: process.env.REACT_APP_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.REACT_APP_FIREBASE_PROJECT_ID,
  storageBucket: process.env.REACT_APP_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.REACT_APP_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.REACT_APP_FIREBASE_APP_ID,
}

const app = initializeApp(firebaseConfig)
const auth = getAuth(app)
const db = getFirestore(app)
const storage = getStorage(app)

export { auth, db, storage }
```

<br>

## Firestore Database Rules

```js
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

```js
service firebase.storage {
  match /b/{bucket}/o {

    // All Paths
    match /{allPaths=**} {
      allow read;
      allow write: if
      isSignedIn() &&
      incomingData().size < 2 * 1024 * 1024 &&
      incomingData().contentType.matches('image/png') || incomingData().contentType.matches('image/jpeg')
    }

    // FUNCTIONS
    function isSignedIn() {
      return request.auth != null;
    }

    function incomingData() {
      return request.resource;
    }
  }
}
```

<br>

## Register User

```js
import { createUserWithEmailAndPassword, updateProfile } from 'firebase/auth'
import { setDoc, doc, getDoc, serverTimestamp } from 'firebase/firestore'
import { auth, db } from '../config/firebase.config'

// on submitting user data
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
await setDoc(doc(db, 'users', user.uid), {
  email,
  name,
  timestamp: serverTimestamp(),
})
```

<br>

## Login User

```js
import { signInWithEmailAndPassword } from 'firebase/auth'
import { auth } from '../config/firebase.config'

// on submitting user data
const userCredential = await signInWithEmailAndPassword(auth, email, password)
```

<br>

## Logout

```js
import { auth } from '../config/firebase.config'
auth.signOut()
```

<br>

## Forgot Password

```js
import { sendPasswordResetEmail } from 'firebase/auth'
import { auth } from '../config/firebase.config'
// on submitting user data
await sendPasswordResetEmail(auth, email)
```

<br>

## Google OAuth

```js
import { signInWithPopup, GoogleAuthProvider } from 'firebase/auth'
import { doc, setDoc, getDoc, serverTimestamp } from 'firebase/firestore'
import { auth, db } from '../config/firebase.config'

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
import { auth, db } from '../config/firebase.config'

const provider = new FacebookAuthProvider()
const result = await signInWithPopup(auth, provider)
const user = result.user
// Check for user
const docRef = doc(db, 'users', user.uid)
const docSnap = await getDoc(docRef)
// If user, doesn't exist, create user
if (!docSnap.exists()) {
  await setDoc(doc(db, 'users', user.uid), {
    email: user.email,
    name: user.displayName,
    timestamp: serverTimestamp(),
    photoURL: user.photoURL,
  })
}
```

<br>

## Private Routes

### useAuthStatus.js

```js
import { useEffect, useState } from 'react'
import { getAuth, onAuthStateChanged } from 'firebase/auth'

export const useAuthStatus = () => {
  const [loggedIn, setLoggedIn] = useState(false)
  const [checkingStatus, setCheckingStatus] = useState(true)

  useEffect(() => {
    const auth = getAuth()
    onAuthStateChanged(auth, user => {
      if (user) {
        setLoggedIn(true)
      }
      setCheckingStatus(false)
    })
  }, [])

  return { loggedIn, checkingStatus }
}
```

### PrivateRoute.js

```js
import { Navigate, Outlet } from 'react-router-dom'
import { useAuthStatus } from '../hooks/useAuthStatus'

const PrivateRoute = () => {
  const { loggedIn, checkingStatus } = useAuthStatus()
  if (checkingStatus) {
    return <h3>Loading...</h3>
  }
  return loggedIn ? (
        <Outlet />
    </>
  ) : (
    <Navigate to='/login' />
  )
}

export default PrivateRoute
```

### PublicRoute.js

```js
import { Navigate, Outlet } from 'react-router-dom'
import { useAuthStatus } from '../hooks/useAuthStatus'

const PublicRoute = () => {
  const { loggedIn, checkingStatus } = useAuthStatus()
  if (checkingStatus) {
    return <h3>Loading...</h3>
  }
  return !loggedIn ? (
        <Outlet />
    </>
  ) : (
    <Navigate to='/' />
  )
}

export default PublicRoute
```

<br>

## Fetch Collections from Database

```js
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

## Fetch Collections from database with Realtime Update

```js
// without query
const getUser = async () => {
  onSnapshot(doc(db, 'users', auth.currentUser.uid), doc => {
    setCurrentUser(doc.data())
  })
}

// with query
const getPosts = async () => {
  const postsRef = collection(db, 'posts')
  const q = query(
    postsRef,
    where('userRef', 'in', doc.data().following),
    limit(15)
  )
  onSnapshot(q, querySnapshot => {
    const posts = []
    querySnapshot.forEach(doc => {
      posts.push({
        id: doc.id,
        data: doc.data(),
      })
    })
    setPosts(posts)
  })
}
```

<br>

## Fetch Document from Collection

```js
import { getDoc, doc } from 'firebase/firestore'
import { getAuth } from 'firebase/auth'
import { db } from '../firebase.config'

//on component load
const docRef = doc(db, 'products', productId)
const docSnap = await getDoc(docRef)
// docSnap to your state after checking it exists
```

<br>

## Fetch Document from Collection with Realtime Update

```js
const getPost = async () => {
  onSnapshot(doc(db, 'posts', postId), doc => {
    setPostData(doc.data())
  })
}

useEffect(() => {
  getPost()
}, [onSnapshot])
```

<br>

## Uploading Images

```js
import {
  getStorage,
  ref,
  uploadBytesResumable,
  getDownloadURL,
} from 'firebase/storage'
import { db } from '../firebase.config'

// Store image in firebase
import { serverTimestamp, addDoc, collection } from 'firebase/firestore'
import { getDownloadURL, getStorage, ref, uploadBytes } from 'firebase/storage'
import { v4 as uuidv4 } from 'uuid'
import { auth, db } from '..//config/firebase.config'

// on upload
const storage = getStorage()
const fileName = `${auth.currentUser.uid}-${acceptedFiles[0].name}-${uuidv4()}`
const storageRef = ref(storage, 'posts/' + fileName)
await uploadBytes(storageRef, acceptedFiles[0])
const photoURL = await getDownloadURL(storageRef)

// add to db
await addDoc(collection(db, 'posts'), {
  image: photoURL,
})
```

<br>

## Create Document in a Collection

```js
import { addDoc, collection, serverTimestamp } from 'firebase/firestore'

//on submit data
const docRef = await addDoc(collection(db, 'products'), formData)
//docRef gives id too if you want to navigate to that
```

<br>

## Delete Document in a Collection

```js
import { addDoc, doc } from 'firebase/firestore'

//on delete
await deleteDoc(doc(db, 'products', productId))
```

<br>

## Update Document in a Collection

```js
import { updateDoc, doc } from 'firebase/firestore'

//on update
await updateDoc(doc(db, 'products'), {
  onSale: false,
})
```

<br>

## Update Array Document in a Collection

```js
import {
  updateDoc,
  doc,
  arrayUnion,
  arrayRemove,
} from 'firebase/firestore'

// add
const handleLike = async () => {
    setSubmitLikeLoading(true)
    await updateDoc(doc(db, 'posts', postId), {
      likes: arrayUnion(auth.currentUser.uid),
    })

// remove
const handleUnlike = async () => {
  setSubmitLikeLoading(true)
  await updateDoc(doc(db, 'posts', postId), {
    likes: arrayRemove(auth.currentUser.uid),
  })
  setSubmitLikeLoading(false)
}
```
