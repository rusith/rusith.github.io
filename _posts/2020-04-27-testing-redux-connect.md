---
layout: post
title: Testing Redux Connected React Components Using Jest
tags: programming javascript react redux testing jest
comments: true
description: Testing React components connected to a Redux store using Jest and React Testing Library.
dateCreated: 2020-04-27
dateModified: 2020-04-27
datePublished: 2020-04-27
dependencies: React
about: Testing React components connected Redux store using Jest and React Testing Library
permalink: /blog/testing-react-redux-connected-components-using-jest
banner: /public/post-data/2020-04-27-testing-redux-connect/banner.png
published: true
---

<img class="banner" src="{{ site.url }}{{page.banner}}">

When writing tests for a React application you might come across the case where you have to test a React
component that is connected to a Redux store using the `connect` function.
Testing these components in isolation might look complicated.
reasons being that the component is wrapped in a HOC so we don't have direct access to the component properties.
and the connect function takes Redux into the picture, so we have a whole layer to mock.

Let's take this simple login component as an example throughout this article.

```tsx
import React, {useEffect} from "react";
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
    const [showError, setShowError] = useState(false);

    useEffect(() => {
        setShowError(!!error);
    }, [error]);

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
            {loading && <img data-testid="loader" alt="Loading" className="loading" />}
            {showError && <p data-testid="errorMessage" className="error">{error}</p>}
            <form onSubmit={handleLogin}>
                <input data-testid="emailInput" required type="email" onChange={(e) => setEmail(e.target.value)} />
                <input data-testid="passwordInput" required type="password" onChange={(e) => setPassword(e.target.value)} />

                <button type="submit" data-testid="loginTrigger">Log In</button>
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

And in the same location, the state,

```tsx
export function login(loginInfo) {
    return {
        payload: loginInfo,
        type: "LOGIN",
    };
}

export interface ILoginState {
    loading: boolean;
    error: string;
}
```


## How to Test

As we are going to test the component, First thing we have to do is to isolate it. 
A Redux based component can communicate in two ways. It can dispatch actions
and it can take store updates as props. So that's what we are going to test. 
It should dispatch the right actions with the right parameters and it should render the data as intended.

### Mocking the Store

To isolate the component, we have to find a way to mock the store and render the component using mock data.
<a href="https://www.npmjs.com/package/redux-mock-store" target="_blank">redux-mock-store</a> is a package that allows us to mock the Redux store.
This will record all actions that are dispatched to the store.

Let's create a helper function that will render the component and return the store and the component itself.

```tsx
function renderComponent(state: ILoginState) {
    const store = mockStore(state);
    return [render((
        <Provider store={store}>
            <LogIn />
        </Provider>
    )), store];
}
```

Now we can use this to render the component inside the Redux provider.

### Testing Dispatch

Let's write our first test. This will test if the component dispatches 
the login action when the inputs are valid.

```tsx
it("should dispatch login action if inputs are valid", () => {
    const [{getByTestId}, store] = renderComponent({ loading: false, error: null});

    fireEvent.change(getByTestId("emailInput"), {
        target: { value: "user_one@gmail.com" },
    });

    fireEvent.change(getByTestId("passwordInput"), {
        target: { value: "password" },
    });

    fireEvent.click(getByTestId("loginTrigger"));

    expect(store.getActions()).toContainEqual(login({ email: "user_one@gmail.com", password: "password" }));
});
```

Here, we submit the form using `fireEvent`. and we expect the list of actions (`getActions()`) of the store 
to contain login action with the right payload. This way we can simulate user behaviors and check if
it calls correct action.

### Testing Props

Most of the time, we want to change the UI based on the input props.
To do this, we can pass any state to the `renderComponent` and write assertions based on the expected behavior.

For example, let's write two tests to see if we only render the loading indicator when `loading` is set to true.

```tsx
it("should show loading indicator when loading is set to true", () => {
    const [{queryByTestId}] = renderComponent({ loading: true } as ILoginState);
    expect(queryByTestId("loader")).toBeTruthy();
});

it("should not show loading indicator when loading is set to false", () => {
    const [{queryByTestId}] = renderComponent({ loading: false } as ILoginState);
    expect(queryByTestId("loader")).toBeFalsy();
});
```

As you can see we just have to pass the current state that we want and write assertions accordingly.

### Testing Effects

The Login component has an effect that checks the prop `error` and set `showError`.
To test this, we have to change the inputs of the component. As our store cannot publish
a new state to the components, what we can do is to re-render the component with the new state.
This will update the props of the component and will run the effect.

Let's write a test for this behavior.

```tsx
it("should show error if an error is available", () => {
    renderComponent({ loading: false, error: null });
    const [{getByTestId}] = renderComponent({loading: false, error: "something went wrong"});

    expect(getByTestId("errorMessage")).toHaveTextContent("something went wrong");
});
```

Below is the complete test file for this component.

```tsx
import "@testing-library/jest-dom/extend-expect";
import {cleanup, fireEvent, render} from "@testing-library/react";
import React from "react";
import {Provider} from "react-redux";
import configureStore from "redux-mock-store";
import LogIn from "./LogIn";
import {ILoginState, login} from "./state";

const mockStore = configureStore([]);
describe("Login", () => {
    function renderComponent(state: ILoginState) {
        const store = mockStore(state);
        return [render((
            <Provider store={store}>
                <LogIn />
            </Provider>
        )), store];
    }

    afterAll(cleanup);

    it("should dispatch login action if inputs are valid", () => {
        const [{getByTestId}, store] = renderComponent({ loading: false, error: null });

        fireEvent.change(getByTestId("emailInput"), {
            target: { value: "user_one@gmail.com" },
        });

        fireEvent.change(getByTestId("passwordInput"), {
            target: { value: "password" },
        });

        fireEvent.click(getByTestId("loginTrigger"));

        expect(store.getActions()).toContainEqual(login({ email: "user_one@gmail.com", password: "password" }));
    });

    it("should show loading indicator when loading is set to true", () => {
        const [{queryByTestId}] = renderComponent({ loading: true } as ILoginState);
        expect(queryByTestId("loader")).toBeTruthy();
    });

    it("should not show loading indicator when loading is set to false", () => {
        const [{queryByTestId}] = renderComponent({ loading: false } as ILoginState);
        expect(queryByTestId("loader")).toBeFalsy();
    });

    it("should show error if an error is available", () => {
        renderComponent({ loading: false, error: null });
        const [{getByTestId}] = renderComponent({loading: false, error: "something went wrong"});

        expect(getByTestId("errorMessage")).toHaveTextContent("something went wrong");
    });
});
```


