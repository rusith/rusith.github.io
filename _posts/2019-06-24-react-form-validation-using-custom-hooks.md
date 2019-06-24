---
layout: post
title: React Form Validation Using Custom Hooks
tags: programming react javascript
comments: true
hasMath: false
---
Validation!
You'll have done it right? 

I have this sign-up form here that I want to validate. What do I need to validate? 
First, firstName and lastName should be entered! and the email should be a valid one. and the passwords should match and should not be empty.
Okay, a typical form with typical validation scenarios. the best example in the world (maybe not)

Here you have the source code for my sweet little simple sign up page. no bullshit at all here. I have removed everything and included only the things that are necessary.

```jsx
import React, { useState } from 'react'


function doSignUp(userInfo) {
  // Go to the server || dispatch an action
}

export default function SignUp() {
  const [firstName, setFirstName] = useState('')
  const [lastName, setLastName] = useState('')
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [repeatPassword, setRepeatPassword] = useState('')


  function handleSignUp() {
    doSignUp({ firstName, lastName, email, password })
  }

  return (
    <div className="sign-up">
      <input onChange={e => setFirstName(e.target.value)} value={firstName} />
      <input onChange={e => setLastName(e.target.value)} value={lastName} />
      <input onChange={e => setEmail(e.target.value)} value={email} />
      <input type="password" onChange={e => setPassword(e.target.value)} value={password} />
      <input type="password" onChange={e => setRepeatPassword(e.target.value)} value={repeatPassword} />

      <button onClick={handleSignUp}>Sign me up! </button>
    </div>
  )
}
```
I have avoided the form tags and stuff for the sake of simplicity.

If you press the button on this page it will go to the signUp process and do the thing. If you haven't done any back end validation, you are screwed up.(always do back end validation. never trust your user (yourself in this case)). We don't want this to happen. so we validate and show sweet little messages in the UI.

```jsx
import React, { useState } from 'react'


function doSignUp(userInfo) {
  // Go to the server || dispatch an action
}

function validateEmail(email) {
    let re = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test(String(email).toLowerCase());
}

function validateForm(values) {
  const errors = {}
  if (!values.firstName) errors.firstName = "First name is required" // pretty standard error messages. cuz im too lazy to think
  if (!values.lastName) errors.lastName = "Last name is required"
  if (!values.email) errors.email = "Email address is required"
  else if (!validateEmail(values.email)) errors.email = "Not a valid email address"
  if (!values.password) errors.password = "Password is required"
  else if (!values.repeatPassword) errors.repeatPassword = "Please repeat the password"
  else if (!values.password != values.repeatPassword) errors.repeatPassword = "Passwords don't match"

  return errors
}

export default function SignUp() {
  const [firstName, setFirstName] = useState('')
  const [lastName, setLastName] = useState('')
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [repeatPassword, setRepeatPassword] = useState('')
  const [errors, setErrors] = useState({})


  function handleSignUp() {
    const errors = validateForm({ firstName, lastName, email, password, repeatPassword })
    setErrors(errors)
    if (!Object.keys(errors).length) {
      doSignUp({ firstName, lastName, email, password })
    }
  }

  return (
    <div className="sign-up">
      <input onChange={e => setFirstName(e.target.value)} value={firstName} />
      {errors.firstName && <p>{errors.firstName}</p>}
      <input onChange={e => setLastName(e.target.value)} value={lastName} />
      {errors.lastName && <p>{errors.lastName}</p>}
      <input onChange={e => setEmail(e.target.value)} value={email} />
      {errors.email && <p>{errors.email}</p>}
      <input type="password" onChange={e => setPassword(e.target.value)} value={password} />
      {errors.password && <p>{errors.password}</p>}
      <input type="password" onChange={e => setRepeatPassword(e.target.value)} value={repeatPassword} />
      {errors.repeatPassword && <p>{errors.repeatPassword}</p>}

      <button onClick={handleSignUp}>Sign me up! </button>
    </div>
  )
}
```

Now if you press the button it won't do anything and show messages below the inputs. good.
you could use something like Formik or some other alternative way to do this validation. But I don't want to load any external library just to validate a few little forms in my app. let's do this ourselves.

Okay, let's see what we can do to improve this code to make it more modular and readable.

You can see the `onChange` handlers in the input have the same repeating pattern. can't we extract it out and reuse that logic?
Let's do that now.

Let's create a hook that will get the value from an event and set it

```jsx

function useInputValue(initialValue) {
  const [value, setValue] = useState(initialValue)

  function retrieveValue(event) {
    setValue(event.target.value)
  }
  return [value, retrieveValue]
}


export default function SignUp() {
  const [firstName, setFirstName] = useInputValue('')
  const [lastName, setLastName] = useInputValue('')
  const [email, setEmail] = useInputValue('')
  const [password, setPassword] = useInputValue('')
  const [repeatPassword, setRepeatPassword] = useInputValue('')
  const [errors, setErrors] = useState({})


  function handleSignUp() {
    const errors = validateForm({ firstName, lastName, email, password, repeatPassword })
    setErrors(errors)
    if (!Object.keys(errors).length) {
      doSignUp({ firstName, lastName, email, password })
    }
  }

  return (
    <div className="sign-up">
      <input onChange={setFirstName} value={firstName} />
      {errors.firstName && <p>{errors.firstName}</p>}
      <input onChange={setLastName} value={lastName} />
      {errors.lastName && <p>{errors.lastName}</p>}
      <input onChange={setEmail} value={email} />
      {errors.email && <p>{errors.email}</p>}
      <input type="password" onChange={setPassword} value={password} />
      {errors.password && <p>{errors.password}</p>}
      <input type="password" onChange={setRepeatPassword} value={repeatPassword} />
      {errors.repeatPassword && <p>{errors.repeatPassword}</p>}

      <button onClick={handleSignUp}>Sign me up! </button>
    </div>
  )
}
```
Okay so far so good. let's now try to remove the repeated state declarations and move them somewhere else. we can create another hook for that.
That will look something like below. This is the main hook that we are going to use to create the form. this includes all the logic to run validations when necessary.

```jsx
function useForm(initialValues, validateForm) {
  if (!initialValues) {
    throw Error('Initial values are required')
  }

  const values = {}
  const valuesWithSetters = {}
  const [errors, setErrors ] = useState({})
  const keys =  Object.keys(initialValues)

  for (let i = 0, l = keys.length; i < l; i ++) {
    const key = keys[i]
    const [val, setVal] = useInputValue(initialValues[key])
    const setValWrapper = (...pars) => {
      if (errors[key]) {
        const er = {...errors}
        delete er[key]
        setErrors(er)
      }
      setVal(...pars)
    }
    valuesWithSetters[key] = [val, setValWrapper, () => errors[key]]
    values[key] = val
  }

  function validate() {
    const errorObject = {}
    validateForm(errorObject, values)
    setErrors(errorObject)
    return Object.keys(errorObject).length < 1
  }

  return  [valuesWithSetters, validate, errors, values ]
}
```

Okay, what are we doing here? Here we pass a set of initialValue and this hook will create a set of internal states for each key in this object with its value. and what the `validate` function does is it will call the validate function that we provide to the hook and set the error messages automatically.

Let's use this in our code. We will also write a Component that will help us to display the error messages. we will also have to change the `validateForm` function to fit with our hook

```jsx
function validateForm(errors, values) {
  if (!values.firstName) errors.firstName = "First name is required" // pretty standard error messages. cuz im too lazy to think
  if (!values.lastName) errors.lastName = "Last name is required"
  if (!values.email) errors.email = "Email address is required"
  else if (!validateEmail(values.email)) errors.email = "Not a valid email address"
  if (!values.password) errors.password = "Password is required"
  else if (!values.repeatPassword) errors.repeatPassword = "Please repeat the password"
  else if (values.password != values.repeatPassword) errors.repeatPassword = "Passwords don't match"
}

function renderErrorMessage(field) {
  return <FieldErrorMessage field={field} >{msg => <p className="error-messge">{msg}</p>}</FieldErrorMessage>
}

export default function SignUp() {
  const [{ firstName, lastName, email, password, repeatPassword }, validate, errors, values ]
    = useForm({ firstName: '', lastName: '', email: '', password: '', repeatPassword: '' }, validateForm)

  function handleSignUp() {
    const valid = validate()
    if (valid) {
      doSignUp(values)
    }
  }

  return (
    <div className="sign-up">
      <input onChange={firstName[1]} value={firstName[0]} />
      {renderErrorMessage(firstName)}
      <input onChange={lastName[1]} value={lastName[0]} />
      {renderErrorMessage(lastName)}
      <input onChange={email[1]} value={email[0]} />
      {renderErrorMessage(email)}
      <input type="password" onChange={password[1]} value={password[0]} />
      {renderErrorMessage(password)}
      <input type="password" onChange={repeatPassword[1]} value={repeatPassword[0]} />
      {renderErrorMessage(repeatPassword)}

      <button onClick={handleSignUp}>Sign me up! </button>
    </div>
  )
}
```

Okay. Now the component looks a bit cleaner. now we can move the hooks that we created into another file and we can also move the helper component (`FieldErrorMessage`) to a separate file so we can use it in other components too.

Let's see how we can re-use these things to create a simple login form.

```jsx
import React, { useState } from 'react'
import { useForm } from './hooks'
import FieldErrorMessage from './FieldErrorMessage'


function doLogin(credentials) {
  // Go to the server || dispatch an action
}

function validateForm(errors, values) {
  if (!values.username) errors.username = "Username is required" // pretty standard error messages. cuz im too lazy to think
  if (!values.password) errors.password = "Password is required"
}

function renderErrorMessage(field) {
  return <FieldErrorMessage field={field} >{msg => <p className="error-messge">{msg}</p>}</FieldErrorMessage>
}

export default function SignUp() {
  const [{ username, password }, validate, errors, values ]
    = useForm({ username: '', password: '' }, validateForm)

  function handleSignUp() {
    const valid = validate()
    if (valid) {
      doLogin(values)
    }
  }

  return (
    <div className="sign-up">
      <input onChange={username[1]} value={username[0]} />
      {renderErrorMessage(username)}
      <input type="password" onChange={password[1]} value={password[0]} />
      {renderErrorMessage(password)}

      <button onClick={handleSignUp}>Sign me in! </button>
    </div>
  )
}
```

And this is our sign up code using hooks imported 

```jsx
import React, { useState } from 'react'
import { useForm } from './hooks'
import FieldErrorMessage from './FieldErrorMessage'


function doSignUp(userInfo) {
  // Go to the server || dispatch an action
}

function validateEmail(email) {
    let re = /^(([^<>()\[\]\\.,;:\s@"]+(\.[^<>()\[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test(String(email).toLowerCase());
}

function validateForm(errors, values) {
  if (!values.firstName) errors.firstName = "First name is required" // pretty standard error messages. cuz im too lazy to think
  if (!values.lastName) errors.lastName = "Last name is required"
  if (!values.email) errors.email = "Email address is required"
  else if (!validateEmail(values.email)) errors.email = "Not a valid email address"
  if (!values.password) errors.password = "Password is required"
  else if (!values.repeatPassword) errors.repeatPassword = "Please repeat the password"
  else if (values.password != values.repeatPassword) errors.repeatPassword = "Passwords don't match"
}

function renderErrorMessage(field) {
  return <FieldErrorMessage field={field} >{msg => <p className="error-messge">{msg}</p>}</FieldErrorMessage>
}

export default function SignUp() {
  const [{ firstName, lastName, email, password, repeatPassword }, validate, errors, values ]
    = useForm({ firstName: '', lastName: '', email: '', password: '', repeatPassword: '' }, validateForm)

  function handleSignUp() {
    const valid = validate()
    if (valid) {
      doSignUp(values)
    }
  }

  return (
    <div className="sign-up">
      <input onChange={firstName[1]} value={firstName[0]} />
      {renderErrorMessage(firstName)}
      <input onChange={lastName[1]} value={lastName[0]} />
      {renderErrorMessage(lastName)}
      <input onChange={email[1]} value={email[0]} />
      {renderErrorMessage(email)}
      <input type="password" onChange={password[1]} value={password[0]} />
      {renderErrorMessage(password)}
      <input type="password" onChange={repeatPassword[1]} value={repeatPassword[0]} />
      {renderErrorMessage(repeatPassword)}

      <button onClick={handleSignUp}>Sign me up! </button>
    </div>
  )
}
```

Certainly, this is not the best solution for this validation problem but its, something that works 😉.