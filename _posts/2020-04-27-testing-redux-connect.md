---
layout: post
title: Testing Redux Connected React Components Using Jest
tags: programming javascript react redux testing jest
comments: true
description: Testing React components connected to a Redux store using Jest.
dateCreated: 2020-04-27
dateModified: 2020-04-27
datePublished: 2020-04-27
dependencies: React
about: Testing React components connected Redux store using Jest and React Testing Library
permalink: /testing-react-redux-connected-components-using-jest
published: true
---

When writing tests for a React application you might come across the case where you have to test a React
component that is connected to a Redux store using the `connect` function.
Testing these components in isolation might look complicated.
reasons being that the component is wrapped in an HOC so we don't have direct access to the component properties.
and the connect function takes Redux in to the picture, so we have a whole layer to mock.

Let's take this simple login component as the example throughout this article.

```tsx
import React from "react";
import {useState} from "react";
import {connect} from "react-redux";
import {ILoginState, login} from "./state";

interface ILoginProps {
    loading: boolean;
    error: string;
}

interface IDispatchProps {
    doLogin: typeof login;
}

type Props = ILoginProps & IDispatchProps;

const Login: React.FC<Props> = ({loading, error, doLogin}) => {
    const [email, setEmail] = useState("");
    const [password, setPassword] = useState("");

    const handleLogin = (e) => {
        const form = e.currentTarget;
        e.preventDefault();
        e.stopPropagation();

        if (form.checkValidity()) {
            doLogin({email, password});
        }
    };

    return (
        <div className="log-in">
            <form onSubmit={handleLogin}>
                <input required type="email" onChange={(e) => setEmail(e.target.value)} />
                <input required type="password" onChange={(e) => setPassword(e.target.value)} />

                <button type="submit">Log In</button>
            </form>
        </div>
    );
};

const mapStateToProps = (state: ILoginState): ILoginProps => ({
    error: state.error,
    loading: state.loading,
});

const mapDispatchToProps = {
    doLogin: login,
};

export default connect(
    mapStateToProps,
    mapDispatchToProps,
)(Login);
```

and in the same location, the state,

```tsx
export function login(loginInfo) {
    return {
        payload: loginInfo,
        type: "SIGN_IN",
    };
}

export interface ILoginState {
    loading: boolean;
    error: string;
}
```


### Mocking the Store

