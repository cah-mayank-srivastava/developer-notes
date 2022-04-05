Next.JS application’s navigational behavior can be put into the following categories:

CASE 1. navigation to a page with server-side rendering, static rendering with prop fetch

CASE 2. events or dispatch triggers on the same page 

CASE 3. static page render with no prop fetch

To supporting universal redux we need to have a separation of concern for the client store and the server store. When we are on the server we want to create a new store every time and if we are on the client-side(after the DOM is painted and React is loaded) we want to use the same client-side store instance. next-redux-wrapper gives access to the HYDRATE action which is dispatched every time server-side props are fetched with the previous state in its payload such that the state values can be persisted. The above three cases can then be handled as:

CASE 1. navigation to a page with server-side rendering, static rendering with prop fetch

1. new server-side store is created on every such page navigation

2. necessary actions are dispatched and the server store is updated

 3. next-redux-wrapper’s HYDRATE action is dispatched which has the server-side store in its action.payload 

4. delta of the server-side store over the previously active state(client-side state) is applied and returned as the updated state 

5. if any client-side store values are supposed to be persisted they should be added back to the updated state in the HYDRATE reducer before returning the new state

CASE 2. events or dispatch triggers on the same page 

 - the wrapper returns the currently active client-side store without any changes as the source of state values, actions dispatched will go through the same client-side store.

To handle CASE 1 and CASE 2 a reducer is written over the combined reducer with all the slices to see if a new server-side store is supposed to be created on navigation. A new store is created in the backend every time to support multiple logins to the application, the store values are not overridden and each user can experience the application uniquely.

 


import { HYDRATE, createWrapper } from "next-redux-wrapper";
import thunkMiddleware from "redux-thunk";

const combinedReducer = combineReducers({
    auth,
    common,
    pharmacies,
});

const reducer = (state, action) => {
// CASE 1: navigation to a page with server-side rendering
    if (action.type === HYDRATE) {
        const nextState = {
            ...state, // use previously active client state
            ...action.payload, // apply delta from hydration
        };

        // preserve store value on client side navigation
        if (state.auth._SESSION_ID) nextState.auth = state.auth;
        if (state.common.users) nextState.common.users = state.common.users;
        if (state.pharmacies.list.length > 0) nextState.pharmacies.list = state.pharmacies.list;

        return nextState;
    } else {
// CASE 2: events or dispatch triggers on the same page
        return combinedReducer(state, action);
    }
};

const initStore = () => {
    return createStore(reducer, bindMiddleware([thunkMiddleware]));
};

export const wrapper = createWrapper(initStore);
CASE 3. static render with no prop fetch

- no actions are dispatched, hence the current client-side state is maintained as the single source of state values
